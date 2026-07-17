# Raspberry Pi host gotchas

Hard-won findings from running NanoClaw on a Pi 4. Each item below caused (or
prolonged) a real outage where **the host looked perfectly healthy and simply
stopped replying**. Apply these when setting up a new Pi.

The failure mode they share: nothing crashes, nothing logs an error, and
`systemctl status` is green. Standard "is it running?" checks all pass while the
agent is silently dead. Don't trust green — verify the specific things below.

## 1. Wi-Fi power save silently blackholes long-lived TCP

**Symptom.** NanoClaw stops replying. The host is up, containers run, scheduled
tasks fire, container kills happen on time — but no messages arrive or send, and
**the error log is completely empty for the whole outage**.

**Cause.** The Pi's onboard Broadcom radio (`brcmfmac`, BCM4345/6) enables Wi-Fi
power save by default. The radio dozes and drops packets with no RST and no ICMP.
Telegram's adapter uses long-polling (`getUpdates`), so its connection just hangs
forever — no exception, no retry, no log line. The adapter waits on a dead socket
believing all is well.

Silence in the error log is the *diagnostic signal*, not the absence of one. A
genuine network drop produces continuous `Network error calling Telegram
getUpdates`. A blackhole produces nothing at all.

**Detect.**

```bash
/usr/sbin/iw wlan0 get power_save     # want: "Power save: off"  (pkg: iw)
dmesg | grep "power save enabled"     # brcmf_cfg80211_set_power_mgmt
```

**Fix.** The driver re-enables power save on **every association**, so a one-shot
at boot is not enough. Two layers:

```bash
# 1. Connection profile (NM re-applies on each activation).
sudo nmcli con modify <wifi-con> 802-11-wireless.powersave 2   # 2 = disable
sudo nmcli con up <wifi-con>     # `device reapply` CANNOT change powersave
```

```sh
# 2. /etc/NetworkManager/dispatcher.d/99-wifi-powersave-off  (chmod 755)
#!/bin/sh
[ "$1" = "wlan0" ] || exit 0
case "$2" in
  up|dhcp4-change|connectivity-change) /usr/sbin/iw dev wlan0 set power_save off || true ;;
esac
```

Notes:
- `nmcli device reapply` fails on this property (`Can't reapply changes to
  '802-11-wireless.powersave'`). A full `con up` bounce is required. Do it from
  the **ethernet** path or you cut your own SSH session.
- A `/etc/NetworkManager/conf.d/*.conf` `[connection] wifi.powersave=2` drop-in
  alone did **not** take effect — set it on the profile.
- If netplan renders NM (connections named `netplan-*`), `nmcli con modify`
  writes through into `/etc/netplan/90-NM-*.yaml` as `wifi.powersave: "2"`, so it
  survives regeneration. Verify it landed there.
- Ethernet is immune to this. A cable is the most robust fix if it's an option.

## 2. RPi OS forces journald to volatile — no post-mortems

**Symptom.** After any crash or reboot, `journalctl -b -1` has nothing.
`/var/log/journal` exists but is empty, which looks like persistence *is* on.

**Cause.** RPi OS ships `/usr/lib/systemd/journald.conf.d/40-rpi-volatile-storage.conf`
with `Storage=volatile`. All kernel/system logs live in RAM and die at reboot.
`/etc/systemd/journald.conf` showing a commented `#Storage=auto` is a red herring
— always check the `/usr/lib` drop-ins:

```bash
systemd-analyze cat-config systemd/journald.conf | grep -i storage
```

**Fix.** `99-` sorts after `40-`, so it wins:

```bash
sudo tee /etc/systemd/journald.conf.d/99-persistent-storage.conf <<'EOF'
[Journal]
Storage=persistent
SystemMaxUse=200M
SystemMaxFiles=10
EOF
sudo systemctl restart systemd-journald
sudo journalctl --flush        # REQUIRED — restart alone leaves it in runtime mode
journalctl --disk-usage        # verify; /var/log/journal/<machine-id>/ should appear
```

Do this on **day one** of a new Pi. Without it the first outage is undiagnosable.

## 3. User units cannot wait for the network

**Symptom.** After a reboot the host is healthy but never replies. Error log
shows a burst of `Telegram setup failed, retrying` then
`ERROR Failed to start channel adapter channel=telegram`. It stays dead until a
manual restart — `Restart=always` does **not** help, because the process didn't
crash; it's running fine with a dead adapter.

**Cause.** NanoClaw's unit is a *user* unit (`systemctl --user`), and it races
Wi-Fi association at boot. The adapter exhausts its ~85s retry budget and gives up
permanently.

**`After=network-online.target` does not fix this.** User units have their own
target namespace and cannot order against system targets — adding it silently
does nothing and looks like a fix. Poll for real connectivity instead:

```ini
[Service]
ExecStartPre=/bin/bash -c 'for i in $(seq 1 30); do getent hosts api.telegram.org >/dev/null 2>&1 && exit 0; sleep 2; done; echo "network not ready after 60s" >&2; exit 1'
```

Failing here is deliberate: it lets `Restart=always` retry until the network is
genuinely up. Keep `ExecStartPre` under `TimeoutStartSec` (default 90s).

The unit is named `nanoclaw-v2-<slug>.service`, **not** `nanoclaw.service` — see
gotcha 5.

## 4. Reading `logs/nanoclaw.log` — two traps

The app's own logfile is append-only on disk and **survives reboots**, so it is
often the only surviving evidence (see gotcha 2). Two ways to misread it:

- **Timestamps have no dates**, and the file spans many days. `grep '^\[19:'`
  silently interleaves every day's 19:00 hour into a fake timeline. Anchor on
  **line numbers** (the file is chronological) or on epoch-ms embedded in ids
  like `msg-1784284675141-...`.
- **It's flagged binary** (ANSI colour codes) — use `grep -a`.

Useful starting point — find the delivery gap, then read around it by position:

```bash
grep -an "Message delivered" logs/nanoclaw.log | tail -5     # gap = outage window
sed -n "${LINE},$((LINE+25))p" logs/nanoclaw.log             # what happened next
grep -an "NanoClaw starting" logs/nanoclaw.log | tail -5     # restart boundaries
```

## 5. Misleading checks (verified false leads)

Things that wasted real time during the outage above:

| Check | Why it misleads |
|-------|-----------------|
| `systemctl --user is-active nanoclaw` | Unit is `nanoclaw-v2-<slug>.service`. systemd reports a **nonexistent** unit as `inactive` — indistinguishable from "stopped". Use `is-enabled` (`not-found`) or list the real name. |
| `vcgencmd get_throttled` after a reboot | Latched bits **reset on power cycle**. `0x0` post-reboot says nothing about the failure and does not exonerate the PSU. |
| `tailscale status` says "offline" | Only means the control plane lost it. `tailscale ping` may still pong via LAN. Not proof the host is down. |
| Solid red / dark green LED | Green ACT blinks on boot-media access; a Pi that is up but network-dead looks identical to a boot failure from across the room. |
| Docker containers healthy | They start at boot independently of the NanoClaw user service. Green containers say nothing about whether NanoClaw is up. |

**Triage order that actually works:** is the *delivery* path moving
(`grep "Message delivered" logs/nanoclaw.log | tail`) → is the host doing local
work during the gap (container kills, sweeps) → if yes, it's a network/adapter
stall, not a crash.

## New-Pi checklist

1. Persistent journald (gotcha 2) — **first**, before anything can break.
2. `sudo apt install -y iw`; disable Wi-Fi power save + dispatcher script (gotcha 1).
3. `ExecStartPre` network wait in the user unit (gotcha 3).
4. `loginctl enable-linger <user>` so the user manager starts at boot.
5. Prefer ethernet where possible; verify `iw wlan0 get power_save` is `off` after
   a reboot **and** after a Wi-Fi reconnect.
