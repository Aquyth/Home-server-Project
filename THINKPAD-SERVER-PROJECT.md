# ThinkPad Minecraft Server — Build Log

> One old laptop → headless Linux Minecraft server → play from my main laptop.
> Single-file project doc. No tutorial, just what I did, why, and what broke.

**Status:** LIVE — server installed, Paper 26.3 running, joined and played
**Date started:** 2026-10-12
**Author:** sharm

---

## 1. Goal

I have a main laptop I play on, and an old ThinkPad that runs okayish. I wanted to do it once and do it best: wipe the old ThinkPad, turn it into a 24/7 Linux server hosting Minecraft, and join from my main laptop on home LAN. Option to add friends outside later without redoing everything.

Constraints I set:
- Do it once, reliable, not again-and-again debugging.
- Old hardware must be respected, no heavy modpacks.
- One doc file for GitHub, first-person, failures included.

## 2. Hardware As-Found

- Machine: Lenovo ThinkPad, i3-4000M era (T440 / T540 / X440 generation)
- CPU: Intel Core i3-4000M @ 2.40 GHz, 2c/4t Haswell
- RAM: 8 GB DDR3 — Windows Memory Diagnostic: **No errors**
- Storage: 168 GB usable — `INTEL SSDSC2BF180A4H` — this is an Intel DC S3500 180GB enterprise SATA SSD
  - CrystalDiskInfo: **Good 84%, 45°C**
  - 84% life left = plenty. Enterprise drive with power-loss protection, ideal for constant world saves. No replacement needed.
  - Partitioned as 97 GB + 70 GB by Windows (C: + D:/Recovery). Will be wiped to one Linux disk.
- Graphics: Intel HD 4600 113 MB — irrelevant, server will be headless.
- Network: WiFi + Ethernet. Ended up on **Ethernet** for stability; WiFi tested at avg 34–94ms with spikes to 1.6s.
- Power: Original charger, AC only. **The battery is dead** — pulling the charger powers it off instantly. So it must stay plugged in, which is why `Power on with AC Attach` matters.

Why this works: i3-4000M single-thread is weak but enough for Paper + 2-8 players vanilla/light plugins. 8 GB lets me give 4 GB to Minecraft and leave rest to OS. S3500 endurance solves the usual old-SSD worry.

Why this won't work: heavy Forge/Fabric modpacks, 20+ players, high view-distance. Not attempting.

## 3. Plan I Locked

- OS: Ubuntu Server 24.04 LTS, wipe, ext4 + LVM, user `minecraft`, OpenSSH only. No desktop, no Docker, no panel — too heavy for i3.
- Server software: Paper 1.21.x (Java Edition). 30-40% faster than vanilla on this CPU.
- Tuned for 8 GB: `-Xms4G -Xmx4G` + Aikar G1GC, `view-distance=7, simulation-distance=6, max-players=10, white-list=on`.
- Network: DHCP reservation `192.168.0.50`, wired. Play via `192.168.0.50:25565`. Later Tailscale for outside friends, no port-forward.
- Ops: systemd auto-restart, nightly tar of `world*` keep 7 days + weekly rsync to main laptop, pre-generate border 4000 once, `fstrim + smartd` for SSD, `tlp + thermald`, lid-ignore.
- Firewall final: `ufw allow 22,25565`. Windows firewall rules do not survive wipe.

### Plan vs Reality (updated during build)

- **Ubuntu Server 24.04 → 26.04 LTS** — 24.04 was superseded; the installer offered 26.04.1.
- **Java 21 → Java 25** — Minecraft 26.1+ requires Java 25+. My original "Java 21" assumption was wrong; the server refused to boot until 25 was installed.
- **Paper 1.21.x → Paper 26.3** — Minecraft moved to year-based versioning. Latest stable build: `26.3-205`.
- **ext4 + LVM → plain ext4** — dropped LVM (one less layer). LUKS kept off deliberately: it would need a password at boot and break auto-power-on.
- **User `minecraft` → admin `sarthak` + service account `minecraft`** — cleaner separation: I log in as `sarthak`, the game runs as the locked `minecraft` user.
- **`white-list=on`** — Paper 26.3 came up with the whitelist enforcing. Added my IGN `Aquyth` and opped it.

## 4. Build Log (chronological)

### 2026-10-12 — Could not open BIOS

Tried Del / F2 like desktop. Failed, booted straight to Windows every time. ThinkPad uses different keys + Fast Startup blocks keypress.

What worked: full Shift+Shutdown (cold, not hibernate), power on, tap `Enter` → got **Startup Interrupt Menu** → pressed `F1` → BIOS Setup opened.

Lesson: on ThinkPad, `Enter → F1` is more reliable than spamming F1. `Fn+F1` if FnLock on.

### 2026-10-12 — No SSD section, no After Power Loss, no VT-x where expected

In BIOS Main I looked for SSD section — nothing. Looked for After Power Loss, Beep at POST, VT-x under Config > CPU — nothing.

Found after searching:
- SSD does not have its own section. Confirmed via `Config > Serial ATA` → `SATA Controller Mode = AHCI` (correct, left as-is) and `Startup > Boot > Boot Priority Order` → `ATA HDD0: INTEL SSDSC2BF...` in 5th position. That *is* the drive. Position 5 fine, will use F12 for USB.
- `Security > Secure Boot = Disabled` already — left it, perfect for Ubuntu USB.
- `Startup > UEFI/Legacy Boot = Both, UEFI First` already — left it. Learned: UEFI is modern boot handoff, Legacy is old. Both+First lets old SSD + new USB both boot.
- `Startup > Boot Mode = Quick` → changed to `Diagnostics`. Quick skips checks/USB detection for 5s faster boot. Diagnostics does full RAM/hardware check, always sees USB. Worth 15s slower for do-once reliability. Other option in menu was `Diagnostics`.
- No `After Power Loss`. Normal — laptop, not desktop. Found equivalent: `Config > Power > Power on with AC Attach` → set `Disabled → Enabled`. What it does: on wall power return after cut, boots itself instead of staying dead. Critical for 24/7.
- No `USB Boot` toggle, no `USB UEFI BIOS Support`. Normal for this BIOS version — USB allowed automatically because USB HDD in boot list. No action.
- `Config > Network > Wake on LAN = AC only` already — left it. What it does: main laptop can wake server over Ethernet when on AC. Free insurance.
- No `Security > Virtualization` VT-x found on this version. Fine — Minecraft doesn't need it.

Saved with `F10 Yes`, rebooted to Windows. Photos taken before changes as backup.

### 2026-10-12 — SSD health check

Opened CrystalDiskInfo Standard on server: **Good 84%, 45°C**.

What it means: SMART self-report, 84% wear left (>20% needed), 0 reallocated implied by Good. 45°C warm but normal for laptop on Windows. Trusted, no replace. Will improve cooling (prop rear, clean fan) for 24/7.

### 2026-10-12 — RAM check, result hidden

Ran `Windows Memory Diagnostic → Restart now`. Rebooted twice, back to Windows with no popup. Thought it failed.

Found result via `Win+R → eventvwr → Windows Logs > System → Ctrl+F → MemoryDiagnostics` (no Find button visible, Ctrl+F worked). Result: **No errors**.

What it means: 8 GB DDR3 has zero bit errors. No silent world corruption risk. Combined with SSD pass, hardware trusted.

### 2026-10-12 — Partitions 97 GB + 70 GB

Concerned split meant problem. Learned: 97+70=~167GB = full 168GB SSD split into C: + D:/Recovery by factory. Normal. Nothing to save on either — confirmed wipe-safe. Will delete both on Ubuntu install to one ext4 disk (~160GB usable).

### 2026-10-12 — Lid + network rabbit hole (longest part)

Set `Control Panel > Power > Lid close = Do nothing (Plugged in), Sleep = Never (Plugged in)`.

Then:
- Server WiFi IP: `192.168.0.5` via `ipconfig`
- Main laptop IP: `192.168.0.9` — same `192.168.0.x` net, good, no hotspot mismatch.
- `ping 192.168.0.5 -t` from main (`-t` = forever until Ctrl+C): **Request timed out** lid closed *and* lid open.
- Server → router `ping 192.168.0.1`: **Reply 100%, 0% loss** — server WiFi up, can talk out.
- Checked server WiFi profile: was `Public` → switched to `Private`. What Public does: coffee-shop mode, blocks all LAN in. Private trusts home LAN. Still Timeout after.
- Plugged Ethernet: new wired IP `192.168.0.3` (WiFi stayed .5). `ping 192.168.0.3`: **Timeout again**.
- Checked wired profile: already `Private`. Still Timeout.
- Logged into TP-Link router at `192.168.0.1` (password on bottom sticker). Searched `Wireless > Wireless Settings` — no AP Isolation there. Found it under `Advanced Settings > AP Isolation` — **unticked / OFF**. So router not isolating. Ruled out.
- Checked main profile: `Private`. Both Private + AP OFF yet Timeout both ways.
- Main → router `ping 192.168.0.1`: **Reply 0% loss**. So both reach router, not each other.
- Checked server `Windows Security > Firewall`: `Private (active)`, Defender managing, no third-party.
- Checked `Windows Defender Firewall with Advanced Security > Inbound Rules > File and Printer Sharing (Echo Request - ICMPv4-In)`: **2 entries, Domain/Private, both Disabled**.

Root cause found: ping switch Off. Even on Private, Windows drops ICMP if rule Disabled. Explains Timeout both ways + `Test-NetConnection 192.168.0.3 -Port 445` → `TcpTestSucceeded False, PingSucceeded False`.

IPv6 question: left IPv6 echo Disabled intentionally. LAN + Paper use IPv4 `192.168.0.x`. No benefit, smaller surface.

Lesson: ping Timeout ≠ Minecraft can't work (Minecraft uses TCP 25565, not ICMP), but both blocked here indicates inbound firewall. WSL question came up here — answered: WSL is 5-min test but eats 2-3GB to Windows, 10-20% CPU to background, Update reboots kill server, NAT complicates LAN. Dedicated Ubuntu is do-once-best.

### 2026-10-12 — WiFi spike test showed lag, staying on WiFi for now per my call

Unplugged Ethernet, tested WiFi-only `192.168.0.5` from main `192.168.0.9` (~5 min):
- `Sent=327, Received=324, Lost=3 (~0.9% loss, Windows rounds to 0%)`
- `Min=3ms, Max=1669ms, Avg=94ms`

Against do-once criteria (pass 0% loss avg <5ms max <50ms) this is laggy — avg 94ms, max 1669ms freeze risk. Noted.

My call: stay on WiFi for now for convenience (desk far from router), accept lag risk, move back to wired `.3` Ethernet later if unplayable. Found real reservation under TP-Link LAN Settings → Address Reservation (not just IP & MAC Binding): reserved `192.168.0.50 → WiFi MAC 54-35-...`. IP & MAC Binding kept as guard (`.50=Bound+Enabled`, stale `.5` deleted). If adapter was set static `.50` earlier, switch back to DHCP/auto so reservation hands `.50` cleanly. Will re-lock to wired MAC on switch. Main inbound left Disabled intentionally (one-way Main→Server Reply enough for Minecraft TCP).

### 2026-10-13 — `.50` reservation finally stuck

TP-Link menus were not where the guides said. The real switch is under **LAN Settings → Address Reservation** (not `Advanced → DHCP`). Reserved `192.168.0.50` to the WiFi MAC. After `ipconfig /release` + `/renew` on the server, it came up as `192.168.0.50` and pinged from main at `0% loss, avg 34ms, max 172ms` — better than the earlier 94ms/1669ms but still WiFi, not wired-fast.

### 2026-10-15 — Installer USB, and a dead Kingston

Tried to flash the Ubuntu ISO with Rufus onto a KINGSTON 32GB stick. First a Windows "write protection" error, then Rufus failed with `Error 0xC003045D: The request could not be performed because of an I/O device error` — even with bad-block check off and a different USB port. Verdict: the Kingston was dead (controller locked read-only). Swapped to a SanDisk Cruzer Blade 32GB; Rufus wrote it fine (`Grub 2.0 MBR`, FAT32, `boot/EFI/casper` present).

Lesson: an I/O error with write-protect complaints on a stick is hardware, not a settings problem. Do not waste time re-formatting a dying USB.

### 2026-10-19 — Ubuntu Server install (wipe)

Booted the USB via `F12`. First attempt hung at the kernel stage on the HD 4600 (blinking cursor, no new lines) — a graphics-mode hang, not a disk problem.

Second attempt, with Ethernet plugged in, went through cleanly:
- Language `English`, keyboard `English (US)`
- Type: `Ubuntu Server` (not minimized), no third-party drivers
- Network: `ENP0S25` (Ethernet) → DHCP → `192.168.0.7/24` — this is why I went wired for the install: WiFi config in the installer hung on the spinner earlier
- Proxy blank, default mirror
- Storage: `Use an entire disk`, **LVM unchecked**, **LUKS off** — wiped the old 97GB + 70GB NTFS partitions into one `ext4` `/` of 166.6GB plus a 1GB EFI partition
- Profile: name `Sarthak`, hostname `thinkserver`, user `sarthak`
- Ubuntu Pro: skipped
- OpenSSH server: **installed**
- Featured snaps: none (kept the base lean; I'll add lab tools deliberately later)

Then `sudo apt update && sudo apt upgrade -y`, and confirmed SSH from the main laptop: `ssh sarthak@192.168.0.7` works. That was the point where the ThinkPad stopped needing a screen.

### 2026-10-19 — Lid still suspended after logind config (fixed with mask)

First tried a `sed` on `/etc/systemd/logind.conf`. It silently did nothing because those `HandleLidSwitch` lines were not in the file. Then added a drop-in:

`/etc/systemd/logind.conf.d/10-lidignore.conf` with `HandleLidSwitch=ignore`, `HandleLidSwitchExternalPower=ignore`, `HandleLidSwitchDocked=ignore`.

Verified with `systemd-analyze cat-config systemd/logind.conf | grep -i lid`. Then closed the lid with SSH open — **connection timed out**. `journalctl -b` showed the truth:

`Lid closed. → Reached target sleep.target → Performing sleep operation 'suspend' → PM: suspend entry (deep)`

So the config was read, but suspend still fired. Fixed properly by masking the sleep targets:

```
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Re-tested with the lid shut: `date` responded, and the lid LED no longer showed sleep. That is the real fix — `ignore` alone did not stop it on this ThinkPad.

Lesson: on old laptops, config that is *read* is not the same as sleep being *prevented*. Mask the target.

### 2026-10-19 — Java, and the Java 21 assumption being wrong

Installed `openjdk-21-jre-headless`. Paper 26.3 refused to start:

> Minecraft 26.1 and newer requires running the server with Java 25 or above.

This was my mistake in the plan — Java 21 is right for MC 1.21.x, but 26.x moved to Java 25. Installed `openjdk-25-jre-headless`; `update-alternatives --config java` already showed Java 25 in auto mode. Left Java 21 installed as a fallback (~200MB, harmless).

### 2026-10-19 — Dedicated service account

Created a locked system user instead of running the game as `sarthak`:

```
sudo useradd -r -s /usr/sbin/nologin -d /opt/minecraft minecraft
sudo chown -R minecraft:minecraft /opt/minecraft
```

`uid=999(minecraft)`, no password, no login shell. From then on server work is done with `sudo -u minecraft /bin/bash`, so a compromised server never lands in an admin shell.

Gotcha: `sudo -u minecraft -s` fails because the account's shell is `nologin`. Use `sudo -u minecraft /bin/bash` instead.

### 2026-10-19 — Paper 26.3 + tuned start script

Downloaded `paper-26.3-205.jar` from the new Paper API (`fill.papermc.io` — the old `api.papermc.io/v2` now returns 410).

Created `/opt/minecraft/survival/start.sh` with `-Xms4G -Xmx4G` and Aikar's G1GC flags, plus `exec java … --nogui`. Accepted the EULA (`eula.txt`).

Tuned `server.properties`: `view-distance=7`, `simulation-distance=6`, `max-players=10`, `spawn-protection=0`, `motd=Thinkserver`.

Side note: I ran `cat paper.jar` by accident and dumped 53MB of binary into the terminal. Harmless; `reset` fixed the screen. Rule learned: `cat` is for text.

### 2026-10-19 — systemd service, firewall, whitelist

Created `/etc/systemd/system/minecraft.service`:

- `User=minecraft`, `Group=minecraft`, `WorkingDirectory=/opt/minecraft/survival`
- `ExecStart=/opt/minecraft/survival/start.sh`
- `Restart=on-failure`, `RestartSec=15`
- `TimeoutStopSec=180`, `KillSignal=SIGTERM`
- `enable --now` so it starts at boot and immediately

Firewall (order matters — SSH allowed before enabling or I'd lock myself out):

```
sudo ufw allow 22/tcp
sudo ufw allow 25565/tcp
sudo ufw enable
```

Whitelist was enforcing out of the box. Because the service runs in the background with no console, I stopped it, ran `./start.sh` in the foreground as `minecraft`, and typed `whitelist add Aquyth` + `op Aquyth` + `stop`. Then `systemctl start minecraft`.

`spawn-protection` explainer for the log: `16` = only ops can edit within 16 blocks of spawn (anti-grief). For a private friends server it's just annoying, so `0`.

### 2026-10-19 — First real session

Joined from the main laptop at `192.168.0.7` in Minecraft `26.3`. Played 32 minutes.

- `/tps` → **20.0 / 20.0 / 20.0**
- `systemctl` reported 4.7G memory peak, ~15min CPU over 32min wall time
- Clean stop exited with `status=143` (128+15 = SIGTERM). World saved fully, but systemd labelled it `failed`.

Fixed the cosmetic failure with `SuccessExitStatus=143` in the unit, so a manual stop reports `inactive (dead)` and `Restart=on-failure` doesn't try to restart after an intentional stop.


## 5. Decisions + Tradeoffs

- Wipe vs dual-boot: wipe. Dual-boot wastes 40GB+, complicates auto-start.
- Ubuntu Server vs Debian: Ubuntu Server 26.04 LTS (24.04 superseded). Guides, `unattended-upgrades`, sane defaults.
- Java 21 vs Java 25: **Java 25**, forced by Minecraft 26.1+. I originally planned 21 and was wrong.
- Paper vs vanilla vs modded: Paper. Vanilla slower on i3, modded needs 8GB+ heap alone.
- Paper vs Fabric: Paper for now (plugins, fastest). Fabric + Lithium/Starlight if I ever want real mods — separate server dir, no reinstall.
- 4G heap vs more: 4G. 8GB total minus OS + cache + buffer. Observed peak 4.7G during play, which is expected for a 4G heap.
- LVM vs plain ext4: plain ext4. One less layer to break; recovery is simpler.
- LUKS encryption: off. Would require a passphrase at every boot and kill auto-power-on.
- Ethernet vs WiFi: Ethernet. WiFi worked but tested at avg 34–94ms with a 1.6s spike — fine for browsing, not for a server.
- Reservation vs static IP: reservation `192.168.0.50` configured on the router; still to be finalised for the Ethernet MAC.
- Tailscale later vs port-forward now: Tailscale. No router holes, same address works.
- Diagnostics boot vs Quick: Diagnostics. 15s slower, catches RAM faults, sees USB.
- Windows firewall: left the main laptop's inbound ping Disabled deliberately. Minecraft only needs Main → Server TCP 25565; the reverse direction was purely diagnostic.

## 6. Result

Working today:

- [x] Ubuntu Server 26.04 installed, Windows fully wiped, SSH from main laptop
- [x] Lid-closed operation — sleep targets masked, verified with lid shut
- [x] Paper 26.3 (build 205) running under systemd, auto-start at boot
- [x] Java 25, 4GB heap, Aikar flags
- [x] Firewall active: 22/tcp + 25565/tcp
- [x] Whitelist + op for `Aquyth`
- [x] Joined from main laptop in Minecraft 26.3 and played 32 minutes
- [x] **TPS 20.0 / 20.0 / 20.0**

Still open:

- [ ] Static/reserved `192.168.0.50` on the Ethernet interface (currently `.7`)
- [ ] Nightly backups + one restore test
- [ ] `fstrim.timer` + `smartd` for the S3500
- [ ] Pre-generate world border 4000
- [ ] Power-cut test (`Power on with AC Attach`)
- [ ] `fwupdmgr` firmware update
- [ ] Tailscale test for friends outside

Limits found: WiFi is not good enough to host on (avg 94ms, max 1669ms spikes). Ethernet is the only sane choice. 8GB is comfortable at 4GB heap for a few players.

## 7. What I'd Do Differently

- Check the Minecraft version **before** installing Java. Java 21 → 25 was a wasted install cycle.
- Check the router's real DHCP reservation page (`LAN Settings → Address Reservation`) instead of trusting guides that point at `Advanced → DHCP`.
- Test the USB stick before planning around it. The Kingston was dead and I found out mid-build.
- On old laptops, don't trust `HandleLidSwitch=ignore` — go straight to masking `sleep.target` and `suspend.target`.
- Photograph every BIOS tab first (I did this, and it helped).
- Keep the `cat` command for text files only.

---

### Assets

- Have: 1 photo — not yet filed. Will rename to `assets/01-first-photo.jpg`.
- Worth capturing for the write-up: BIOS Boot Priority, CrystalDiskInfo 84%, `journalctl` suspend lines before/after mask, `ufw status`, `/tps` 20/20/20, the in-game join screen.

> Secrets redacted: no router admin password, no public IP, no whitelist UUIDs.
