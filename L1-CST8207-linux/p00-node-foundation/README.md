# DC-FOUNDATION-01 — Raspberry Pi Node Foundation

### Lab Execution Record & Meta Report

**Project ID:** P00 (Foundation)
**Node:** `dc-node-01`
**Platform:** Raspberry Pi 5 Model B Rev 1.1 (8 GB) · Raspberry Pi OS 64-bit (Debian 13 "Trixie") · kernel 6.18.34+rpt-rpi-2712
**Author:** Redji Jean Baptiste
**Tier:** T2 (Standard Lab Report)
**Build window:** 2026-07-09 → 2026-07-22
**Status:** 🟢 Part A steps 0–12 complete, evidenced, and remediated · **SSH hardening proven by negative test (Fig 59)** · reboot persistence verified · **D1–D8 all closed** · remaining: portfolio push (D9) · node reachable at static `172.20.10.10`

---

> **How to read this document.**
> **Part A** is the reproducible build manual — what to do, in order, each step ending in a `✓ Verify`.
> **Part B** is the meta report — the portfolio deliverable, written against the evidence captured while executing Part A.
> **Part 0** is the pre-flight the original manual assumed was already done. It is documented because that is where the platform decisions actually get made, and two of them diverged from plan.
>
> Where a step failed, the failure is shown with its evidence and its fix. The build was rebuilt once from scratch (§3.4) after a self-inflicted lockout. That episode is the most instructive material in the report and is documented at full length rather than smoothed over.

---

## Part 0 — Image preparation and first contact

### 0.1 Tooling

| Tool | Version | Role |
|---|---|---|
| Raspberry Pi Imager | v2.0.10 (Windows 11 Home 26200) | Writes the OS image and injects first-boot customisation |
| PuTTY | current | SSH client from the Windows workstation |
| OpenSSH for Windows | built-in | `ssh`, `scp`, `ssh-keygen` from PowerShell (used from §4 onward) |

### 0.2 Device selection

**Raspberry Pi 5** was selected — the entry covering Pi 5, 500/500+, and Compute Module 5.

> **Fig 0.1** — Imager: *Select your Raspberry Pi device* → Raspberry Pi 5

This is not cosmetic. The Pi 5 uses the BCM2712 SoC and a different boot chain than the Pi 4; selecting the wrong device offers images whose kernel and firmware will not boot the board.

### 0.3 Operating system selection

**Raspberry Pi OS (64-bit)** — the recommended build, "a port of Debian Trixie with the Raspberry Pi Desktop", released **2026-06-18**, already **cached on the computer**.

> **Fig 0.2** — Imager: *Choose operating system* → Raspberry Pi OS (64-bit), Trixie, released 2026-06-18

**⚠ Deviation from the manual.** The manual specifies **Bookworm (Debian 12)**. The current recommended image is **Trixie (Debian 13)**; Bookworm is now the *Legacy* entry in the same list. Trixie was accepted deliberately — it is the vendor-recommended build and receives current security updates. The consequence is that every "Bookworm" reference downstream must be read as Trixie, and any release-specific instruction re-verified rather than trusted. See §8.1.

64-bit was chosen over 32-bit because the board carries 8 GB of RAM — a 32-bit userland cannot address it coherently per process — and because the container images this node will run are predominantly `arm64` (confirmed later in Fig 43: Docker pulled the `arm64v8` variant).

### 0.4 Target storage

The only eligible target was a **59.5 GB USB Mass Storage Device**, mounted as `D:\`.

> **Fig 0.3** — Imager: *Select your storage device* → Mass Storage Device USB Device, 59.5 GB, mounted as `D:\`

Two things in this screenshot matter:

1. **A Windows AutoPlay notification appeared** — *"bootfs (D:) There's a problem with this drive. Scan the drive now and fix it."* Expected, and must be **ignored**. Windows cannot read the Linux `ext4` root partition and misreports the disk as damaged. Running the Windows repair tool against a freshly written Pi image is a genuine way to corrupt it.
2. **The target is a card reader, not an SSD.** Not visible as a problem until `df -h` ran on the booted node. See §8.2.

### 0.5 First-boot customisation

The Imager writes these settings into the boot partition so the node comes up configured on its first boot with no monitor or keyboard attached. This is what makes the build headless and repeatable.

| Pane | Value set | Rationale |
|---|---|---|
| Hostname | `dc-node-01` | Fleet naming convention: `dc` = project, `node` = role class, `01` = ordinal |
| Localisation | Ottawa (Canada) · America/Toronto · `us` | Correct local time is a prerequisite for readable logs, valid TLS, and TOTP later |
| User | `admin` | Non-root administrative account, created at flash time |
| Wi-Fi | SSID + WPA2 passphrase | The node's only link at first boot |
| Remote access | **SSH enabled**, password authentication | Bootstrap access only; hardened to key-based in §4 |

> **Fig 0.4** — Customisation: hostname `dc-node-01`
> **Fig 0.5** — Customisation: Localisation — Ottawa (Canada) / America/Toronto / us
> **Fig 0.6** — Customisation: Wi-Fi — SSID entry with validation error
> **Fig 0.7** — Customisation: Remote access — Enable SSH, password authentication

**Fig 0.6 captures a real validation failure rather than a clean pane:** the Imager rejected the passphrase with *"Password is too short (min 8 characters)"* and greyed out **NEXT**. This is WPA2-PSK's own floor — the standard defines pre-shared keys as 8–63 ASCII characters. The passphrase was re-entered at full length and the pane validated. Worth keeping: the Imager enforces this *before* writing, which is far cheaper than discovering a non-joining node after first boot.

**Note on account naming.** The manual's Part B references a user `redji`; the account created is **`admin`**. `admin` was kept — role-descriptive naming outlives person-descriptive naming on shared infrastructure. See §8.4, and D2 in §13 for the loose end this later produced.

### 0.6 Write and verify

> **Fig 0.8** — Imager: *Write image* summary — Pi 5 · Raspberry Pi OS (64-bit) · Mass Storage Device USB Device; five customisations listed
> **Fig 0.12** — Imager: *Write complete!* — all five customisations confirmed applied; "The storage device was ejected automatically."

Reading the summary pane before committing is the cheapest verification in the build. The completion pane restates the same five items with checkmarks, which is the confirmation that customisation was *written*, not merely *entered*.

> **Note.** Fig 0.12 is from the **second** imaging pass (§3.4). The wizard is identical both times, which is itself the point: the recovery was not improvisation, it was the same reproducible procedure run again.

### 0.7 First contact over SSH

With the card in the Pi and the board powered, the node joined Wi-Fi and registered via mDNS. **No monitor or keyboard was ever attached to the Pi.**

> **Fig 0.9** — PuTTY configuration, empty session
> **Fig 0.10** — PuTTY: Host Name `dc-node-01`, Port 22, SSH
> **Fig 0.11** — Session open: `login as: admin` against `dc-node-01.local`

The `.local` suffix resolves through **mDNS/Avahi**, enabled by default on Raspberry Pi OS. This is what makes the node findable before it has a known address — and it is exactly why a static address is still required: mDNS is a convenience for humans on the same L2 segment, not a dependable address for services. It also proved to be the recovery path that saved the rebuild (§3.4).

**✓ Verify:** password accepted; shell obtained as `admin@dc-node-01`.

---

# PART A — THE SETUP MANUAL (execution record)

## 0. Baseline capture

```bash
hostnamectl; uname -a; ip a; df -h; cat /proc/device-tree/model
```

### 0a. System identity — **Fig 1** (`hostnamectl`)

```
 Static hostname: dc-node-01
      Machine ID: c8624351abc9461599107368737901 7f
Operating System: Debian GNU/Linux 13 (trixie)
          Kernel: Linux 6.18.34+rpt-rpi-2712
    Architecture: arm64
```

The hostname set in the Imager survived to the running system — first-boot customisation worked. `-rpi-2712` is the Pi 5 SoC-specific kernel; `arm64` confirms the 64-bit userland. **Record the Machine ID** — it becomes the proof of rebuild in §3.4.

### 0b. Hardware — **Fig 2** (`cat /proc/device-tree/model`)

```
Raspberry Pi 5 Model B Rev 1.1
```

Read from the device tree the firmware handed the kernel — the board's own claim about itself, not a guess from a label.

### 0c. Network baseline — **Fig 3** (`ip a`)

| Interface | State | Address |
|---|---|---|
| `lo` | UP | `127.0.0.1/8` |
| `eth0` | **NO-CARRIER, DOWN** | — |
| `wlan0` | UP, LOWER_UP | `10.0.0.249/24` **dynamic**, `valid_lft 172252sec` |

The node is reachable **only over Wi-Fi**. `eth0` is up administratively but has no carrier — nothing plugged in. The IPv4 address carries `dynamic` and a finite `valid_lft` (≈2 days), confirming a **DHCP lease, not a reservation**. That lease is the single most important finding in this baseline: it is the thing §3 exists to remove.

### 0d. Storage — **Fig 4** (`df -h`)

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  6.8G   49G  13% /
/dev/mmcblk0p1  505M   87M  418M  18% /boot/firmware
udev            3.9G     0  3.9G   0% /dev
```

**✗ Verify FAILED against the manual.** The manual requires *"`df -h /` shows your SSD … not `mmcblk0`."* Root is on **`/dev/mmcblk0p2`** — `mmcblk` is the kernel's multimedia-card block device. **The node boots from microSD.** The 59.5 GB "USB Mass Storage Device" of Fig 0.3 was a **USB card reader with the microSD inside**; the card was then moved into the Pi's own slot, where it enumerates as `mmcblk0`.

The build continued on microSD deliberately — every subsequent step is storage-agnostic. The consequence is real and is carried into §8.2 and §9 rather than dropped: microSD has markedly lower random-write endurance than NVMe, which matters for a node that will run containers and write logs continuously.

Also visible: `udev` at **3.9 G**, consistent with an 8 GB board; root **49 G available at 13 %**, already expanded to fill the card — so the manual's *Expand Filesystem* step is a no-op here.

---

## 1. Full system update and firmware

```bash
sudo apt update && sudo apt full-upgrade -y
sudo rpi-eeprom-update -a
sudo reboot
```

> **Fig 5** — first `apt update && full-upgrade`

```
Hit:1 http://deb.debian.org/debian trixie InRelease
Hit:2 http://deb.debian.org/debian trixie-updates InRelease
Hit:3 http://deb.debian.org/debian trixie-security InRelease
Hit:4 http://archive.raspberrypi.com/debian trixie InRelease
60 packages can be upgraded.
Upgrading: chromium  chromium-common  cups  cups-client  libinput10 …
```

Four repositories responded — Debian base, `-updates`, `-security`, and Raspberry Pi's own archive, all on `trixie`. **60 packages** were pending on an image only weeks old, which is the practical argument for §7: a node left alone drifts out of patch currency within days. `full-upgrade` was used rather than `upgrade` because it permits package removal to resolve dependency changes — the correct choice where a stale held-back package is worse than a removed one.

> **Fig 16** — post-rebuild repeat: `sudo apt update && supd apt full-upgrade -y` → `-bash: supd: command not found`; **52 packages can be upgraded**

A typo (`supd` for `sudo`) in the second half of the chain. Because `&&` runs the right side only if the left side succeeded, `apt update` completed normally and only the upgrade was skipped — the shell reported the failure plainly and nothing was left half-done. Harmless, and worth logging as T7: **read the tail of a chained command, not just the head.**

> **Fig 17** — `sudo rpi-eeprom-update -a`, then `sudo reboot`
> **Fig 18** — after reboot, `sudo rpi-eeprom-update -a`

```
BOOTLOADER: up to date
   CURRENT: Tue 26 May 15:01:25 UTC 2026 (1779807685)
    LATEST: Tue 26 May 15:01:25 UTC 2026 (1779807685)
   RELEASE: default (/usr/lib/firmware/raspberrypi/bootloader-2712/default)
```

**✓ Verify PASSED.** `CURRENT` equals `LATEST` and the bootloader reports **up to date** — checked *after* the reboot, which is the only check that means anything, since an EEPROM update is staged and applied at boot. `bootloader-2712` again confirms Pi 5 firmware.

---

## 2. Base configuration — `raspi-config`

```bash
sudo raspi-config
```

The tool opens with the board's own identification in the title bar — **`Raspberry Pi 5 Model B Rev 1.1, 8GB`** — independently corroborating Fig 2 and confirming the full 8 GB is visible to firmware.

> **Fig 6** — main menu · **Fig 7** — System Options · **Fig 8** — hostname prompt · **Fig 9** — Localisation Options

**Fig 7** lists *"S3 Password — Change password for the 'admin' user"*: `raspi-config` has picked up the account created at flash time and names it explicitly. Further confirmation the Imager's customisation landed.

**Fig 8** is the payoff: the hostname field arrives **already containing `dc-node-01`**. Nothing needed changing. Setting it in the Imager collapsed the manual's §2 hostname step from a configuration task into a **verification** task — which is precisely the property a reproducible build should have.

**✓ Verify:** hostname `dc-node-01`, corroborated three ways — `hostnamectl` (Fig 1), the shell prompt `admin@dc-node-01`, and `raspi-config` (Fig 8).

| Remaining item | Path | Status |
|---|---|---|
| Expand Filesystem | Advanced Options | **N/A** — Fig 4 shows 58 G root already spanning the card |
| Boot Order | Advanced Options | Deferred — relevant only on migration to SSD (§8.2) |
| GPU Memory → 16 | Performance Options | Deferred — headless server needs no GPU RAM |

---

## 3. Static addressing — the failure, the lockout, and the rebuild

*This is the longest section in the report because it is the only one where the build actually broke. Four distinct faults occurred in sequence, each with a different root cause and a different lesson.*

### 3.1 Discover before you modify — **Fig 10**

```bash
nmcli device status
nmcli connection show
```

```
DEVICE          TYPE       STATE                   CONNECTION
wlan0           wifi       connected               netplan-wlan0-Calypso
eth0            ethernet   unavailable             --

NAME                    UUID                                  TYPE      DEVICE
netplan-wlan0-Calypso   9ea2cb7d-3177-3255-a3ac-23adf3a4e7ad  wifi      wlan0
netplan-eth0            75a1216a-9d1a-30cd-8aca-ace5526ec021  ethernet  --
```

Three readings:

- The active profile is **`netplan-wlan0-Calypso`**, *not* the manual's `"Wired connection 1"`. Running the manual's command verbatim fails with *unknown connection*. **Connection names are discovered, never assumed.**
- The `netplan-` prefix reveals a **configuration layer above the tool being used**: Netplan renders the profiles, NetworkManager is the backend. A change made only with `nmcli` can be reverted when Netplan re-renders.
- `eth0` is `unavailable` — profile exists, no carrier, consistent with Fig 3.

### 3.2 Fault 1 — an unterminated quote — **Fig 11**

```bash
admin@dc-node-01:~ $ sudo nmcli connection modify "netplan-wlan0-Calypso
> "
Error: unknown connection 'netplan-wlan0-Calypso
'.
```

The closing `"` was omitted. Bash printed its **continuation prompt `>`** — the shell was still reading the string — and the newline was swallowed *into the argument*. The connection name handed to `nmcli` was literally `netplan-wlan0-Calypso\n`, which matches nothing.

The diagnostic tell is in the error message itself: the closing `'.` appears **on its own line**, because the embedded newline is being echoed back. Learning to read that is worth more than the fix, because the same signature appears in every quoting bug.

**Lesson:** a `>` prompt where you expected a command prompt means the shell is still parsing. `Ctrl-C` and retype — never keep typing into it.

### 3.3 Fault 2 — the right command, the wrong subnet — **Figs 12 → 13**

Re-run with correct quoting, and it was accepted without error:

```bash
sudo nmcli connection modify "netplan-wlan0-Calypso" \
  ipv4.addresses 192.168.1.50/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "1.1.1.1,9.9.9.9" \
  ipv4.method manual
```

**The command succeeded. The node was lost.**

> **Fig 13** — `PuTTY Fatal Error: Network error: Software caused connection abort`

The values are the manual's **literal example values**. This node is on **`10.0.0.0/24`** (Fig 3). Assigning `192.168.1.50/24` moved the interface to a subnet that does not exist on this LAN: the node had no route to the workstation, the workstation had no route to the node, and the established SSH session was severed mid-flight.

This is the most important single result in the report, for three reasons:

1. **`nmcli` returned silently.** It validated the *syntax* of the address, not its *reachability*. A command exiting zero is not evidence that a step worked — this is the concrete case for the manual's `✓ Verify` discipline.
2. **The lockout came through the network, not through SSH.** The manual's §13 anticipates being locked out by disabling password auth. The actual lockout arrived one step earlier and by a completely different mechanism. Hardening checklists tend to guard the door they expect to be attacked.
3. **Copying example values is the underlying error.** `192.168.1.x` is the manual's illustration, not this network's addressing. Every address in a build manual is a placeholder until checked against `ip a` on the actual node.

> **Fig 14** — Administrator PowerShell: page after page of `169.254.x.x  dynamic`

Diagnosis from the workstation. `169.254.0.0/16` is **APIPA/link-local** — what a host self-assigns when **no DHCP server answers**. Hundreds of such entries in the neighbour cache confirm the workstation's own interface had fallen back to link-local: the segment was not handing out addresses, and the Pi was not going to be found on it. This distinguishes *"the Pi is misconfigured"* from *"this whole segment has no DHCP"* — a distinction that determines whether you fix the node or fix the network.

### 3.4 Recovery — rebuild, and the proof it happened

Recovery was by **re-imaging the microSD** using the identical Imager procedure of Part 0 (**Fig 0.12**, *Write complete!*). With `eth0` unconnected and no monitor attached, no in-place repair path existed — this is exactly the narrowed recovery surface predicted in §8.3, arriving sooner than expected.

> **Fig 15** — post-rebuild `hostnamectl`

```
 Static hostname: dc-node-01
      Machine ID: 9e98c91dd1114cc9be486ec5fe920aea     ← was c8624351abc946159910736873790117f
Operating System: Debian GNU/Linux 13 (trixie)
          Kernel: Linux 6.18.34+rpt-rpi-2712
```

**The Machine ID changed.** `/etc/machine-id` is generated on first boot of a fresh installation, so a changed value is unforgeable evidence that this is a **new install, not a repaired one**. The hostname, OS, and kernel are identical — the customisation is reproducible; the identity is not. That contrast is what makes this a *rebuild* rather than a *rollback*, and it is worth stating in a portfolio: the build was reproducible enough that losing the node cost one imaging pass, not a day.

### 3.5 The static address, applied correctly — **Figs 19 → 21**

> **Fig 19** — after `sudo raspi-config`, `nmcli device status`:

```
DEVICE   TYPE   STATE        CONNECTION
wlan0    wifi   connected    Calypso        ← was netplan-wlan0-Calypso
```

**The profile is now named `Calypso` — the `netplan-` prefix is gone.** The reflash regenerated the connection outside Netplan's rendering, so NetworkManager is now the sole owner of this profile. This **closes the risk logged as T5**: there is no longer an upper layer that can silently re-render and revert an `nmcli` change. It also re-proves §3.1's rule — the name changed between two runs of the same build, so it had to be re-discovered.

> **Fig 20** — `sudo nmcli conneccction modify …` → `Error: argument 'conneccction' not understood. Try passing --help instead.`

Fault 3: a typo in the subcommand. Unlike Fault 2, this failed **loudly and harmlessly** — `nmcli` refused to guess. Contrast is the lesson: the command that broke the node (Fig 12) is the one that produced *no* error. **Noisy failures are cheap; silent successes are expensive.**

Corrected, with this network's real values:

```bash
sudo nmcli connection modify "Calypso" \
  ipv4.addresses 10.0.0.2/24 \
  ipv4.method manual \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "8.8.8.8 8.4.4.4"
```

> **⚠ Defect D1 — open.** `8.4.4.4` is a **typo for `8.8.4.4`** (Google Public DNS secondary). `8.4.4.4` is a different, unrelated address. Because `8.8.8.8` is listed first and answers, resolution works and **the fault is invisible in normal operation** — it surfaces only when the primary is unreachable, i.e. exactly when the secondary is needed. Fix:
> ```bash
> sudo nmcli connection modify "Calypso" ipv4.dns "8.8.8.8 8.8.4.4"
> sudo nmcli connection up "Calypso"
> ```

> **Fig 21** — `ip a` confirming the result

```
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> state UP
    inet 10.0.0.2/24 brd 10.0.0.255 scope global noprefixroute wlan0
       valid_lft forever preferred_lft forever
```

**✓ Verify PASSED.** Compare against the Fig 3 baseline, which is the whole point of having captured it:

| | Baseline (Fig 3) | After (Fig 21) |
|---|---|---|
| Address | `10.0.0.249/24` | `10.0.0.2/24` |
| Flags | `dynamic` | `noprefixroute` — **no `dynamic`** |
| Lifetime | `valid_lft 172252sec` | `valid_lft forever` |

`valid_lft forever` with no `dynamic` flag is the definitive signature of a **manually assigned** address. `noprefixroute` means NetworkManager did not auto-install a subnet route for it, having installed routing explicitly instead — normal for a manual profile.

### 3.6 Fault 4 — the node outran its static address — **Figs 22 → 23**

The node was later physically relocated. The evidence:

> **Fig 22** — `ssh admin@192.168.1.2` succeeds; `ip a` shows **`eth0` UP with `192.168.1.2/24`**, `wlan0` **NO-CARRIER, DOWN**
> **Fig 23** — later still: `eth0` UP, `wlan0` UP with **`172.20.10.4/28`, `dynamic`, `valid_lft 3423sec`**

Three networks in three sessions:

| Session | Link | Address | Type |
|---|---|---|---|
| Fig 21 | Wi-Fi `wlan0` | `10.0.0.2/24` | static |
| Fig 22 | Wired `eth0` | `192.168.1.2/24` | DHCP |
| Fig 23 | Wi-Fi `wlan0` | `172.20.10.4/28` | DHCP |

`172.20.10.0/28` is the **iOS Personal Hotspot** range — a 14-usable-host `/28` handed out by a phone. The `3423sec` lease is short and the range is tiny, which is why a hotspot is a workable bootstrap link and a poor server home.

**The lesson is structural, not clerical.** A static address is **bound to one subnet**. Move the node and it becomes either wrong or irrelevant: on the wired segment DHCP took over `eth0` entirely, and the carefully-set `10.0.0.2` on `wlan0` became inert. §3.3 proved a static address on the wrong subnet disconnects you; §3.6 proves a static address is only as durable as the node's location.

**Consequence for the fleet convention.** A node that moves cannot be addressed statically at the host. The durable answer is a **DHCP reservation on the router**, keyed to the interface MAC (`88:a2:9e:c2:77:d7` for `eth0`, `88:a2:9e:c2:77:d8` for `wlan0` — both in Fig 3 and stable across the rebuild). The node then keeps one address on its home network without carrying a configuration that breaks everywhere else. Recorded as a design change for the permanent installation.

> **Fig 24** — `df -h` re-verified after the rebuild: `/dev/mmcblk0p2  58G  6.8G  49G  13% /`

Identical to Fig 4. The rebuild reproduced the same filesystem layout — and the same microSD limitation (§8.2) — confirming nothing about the storage posture changed.

---

## 4. Users, SSH keys, and daemon hardening

### 4a. Group membership and account management — **Figs 25 → 26**

```bash
admin@dc-node-01:~ $ groups
admin adm dialout cdrom sudo audio video plugdev games users netdev lpadmin gpio i2c spi render input
```

**✓ Verify:** `sudo` is present — the flashed account is a full administrative user, so no new admin account is required. The remaining groups are the Raspberry Pi OS defaults granting hardware access (`gpio`, `i2c`, `spi`), device access (`dialout`, `plugdev`), and printing administration (`lpadmin`).

> **Fig 25** — user management sequence

```bash
sudo adduser redji            # fatal: The user `redji' already exists.
sudo usermod -aG sudo redji
passwd redji                  # passwd: You may not view or modify password information for redji.
sudo passwd redji             # passwd: password updated successfully
```

Three separate teaching points in five lines:

1. **`adduser` is idempotent-by-refusal** — it will not silently clobber an existing account.
2. **`passwd redji` failed; `sudo passwd redji` succeeded.** This is the Linux permission model demonstrated cleanly: an unprivileged user may change only their own password, because `/etc/shadow` is root-owned and readable only by root. The error text is a *correct* refusal, not a malfunction.
3. **`usermod -aG` — the `-a` matters.** Without `-a`, `-G` *replaces* the entire supplementary group list rather than appending, which is a classic way to strip a user of `sudo` while trying to grant it.

> **⚠ Defect D2 — open.** `redji` **already existed** but was never created in this build, and §0.5 records `admin` as the only account set at flash time. Its origin is unexplained. It was then **granted `sudo` and given a new password** — an account of unverified provenance was escalated to full administrative privilege. On a node whose whole premise is least privilege, that is a real finding. Resolve before the portfolio push:
> ```bash
> getent passwd redji          # UID, home, shell — was it flash-created or hand-made?
> sudo lastlog -u redji        # has it ever logged in?
> sudo passwd -l redji         # lock it if it is not needed
> ```

> **Fig 26** — `sudo adduser testuser1` → password set, GECOS *Full Name: Test User*, `Is the information correct? [Y/n] y`

A deliberate account-lifecycle demonstration for **CST8207**: interactive `adduser` creating the home directory, setting the password, and collecting GECOS fields. `testuser1` is a lab artifact and should be removed before the node goes into service:
```bash
sudo deluser --remove-home testuser1
```

### 4b. Key generation and distribution — **Figs 27 → 28**

Keys were generated **on the Windows workstation** and the public half pushed to the node — the correct direction. A private key should never leave the machine that generated it.

> **Fig 27** — PowerShell: `arp -a`, key inventory, and `scp`

```powershell
PS C:\Users\jredj\.ssh> ls
  399  id_ed25519          7/11/2026 11:40 PM
   92  id_ed25519.pub      7/11/2026 11:40 PM
 3369  id_rsa              3/7/2026  6:06 AM
  736  id_rsa.pub          3/7/2026  6:06 AM
 2763  known_hosts
 1995  known_hosts.old

PS C:\Users\jredj\.ssh> scp .\id_ed25519.pub admin@172.20.10.10:~/.ssh
The authenticity of host '172.20.10.10' can't be established.
ED25519 key fingerprint is SHA256:KjzNAN5t3U+mk1ZX03Ot9eteOZ1kctbTYCWQF+SBnpk.
Are you sure you want to continue connecting? yes
Warning: Permanently added '172.20.10.10' (ED25519) to the list of known hosts.
id_ed25519.pub                                   100%   92   6.4KB/s   00:00
```

Notes:

- **Ed25519 over RSA.** Both key types are present; the Ed25519 pair (`399`/`92` bytes vs RSA's `3369`/`736`) is the one deployed. Ed25519 gives ~128-bit security in a fraction of the size, with faster verification and no parameter-choice footguns.
- **The host-key prompt is a security control, not a formality.** On first connection there is no cached fingerprint, so SSH asks the human to vouch for it. Accepting blindly is how a first-connection MITM succeeds. The fingerprint `SHA256:KjzNAN5t…` should be compared against `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` run on the node's console.
- **`known_hosts.old`** exists alongside `known_hosts` — a rewrite happened, consistent with the node's identity changing when it was re-imaged (§3.4) and its address changing repeatedly (§3.6).
- The `arp -a` output above the transfer shows the workstation holding **three** interfaces — `192.168.1.0`, `169.254.83.107` (APIPA again), and `192.168.240.1` (a hypervisor virtual switch). Multi-homing like this is a common reason "the Pi is unreachable" — traffic can leave via the wrong interface entirely.

> **Fig 28** — on the node, installing the key

```bash
admin@dc-node-01:~ $ ls -a          # .ssh present, alongside .sudo_as_admin_successful
admin@dc-node-01:~ $ cd ./.ssh
admin@dc-node-01:~/.ssh $ ls
authorized_keys  id_ed25519.pub
admin@dc-node-01:~/.ssh $ cat ./id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEmoiuVRdTIjB4pNNOJo4Pzp+V2p7wGiegB/saUDnekR jredj@MSI
admin@dc-node-01:~/.ssh $ cat ./id_ed25519.pub > ./authorized_keys
admin@dc-node-01:~/.ssh $ cat ./authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEmoiuVRdTIjB4pNNOJo4Pzp+V2p7wGiegB/saUDnekR jredj@MSI
```

The key comment `jredj@MSI` identifies the workstation of origin — which is the entire practical value of the comment field when auditing `authorized_keys` later.

> **⚠ Practice note — `>` versus `>>`.** `cat … > ./authorized_keys` **truncates and replaces** the file. Here that was harmless — the file was effectively empty and there is one key. On any node with existing authorized keys it **silently revokes every one of them**, which on a password-disabled host is an instant lockout. The habit to build is `cat key.pub >> ~/.ssh/authorized_keys`, or better `ssh-copy-id`, which appends and fixes permissions in one step.
>
> **Permissions were never verified.** `sshd` **silently ignores** `authorized_keys` if the file or `~/.ssh` is group- or world-writable — a failure mode that presents as "the key just doesn't work" with nothing useful in the client output. Add to the checklist:
> ```bash
> chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
> ls -la ~/.ssh          # expect drwx------ and -rw-------
> ```

### 4c. Daemon configuration — **Figs 29 → 31**

> **Fig 29** — `nano /etc/ssh/sshd_config.tmp`, showing the region around the auth directives

```
# To disable tunneled clear text passwords, change to "no" here!
#PasswordAuthentication yes
#PermitEmptyPasswords no
```

> **Fig 31** — a syntax check catching a real error

```
AcceptEnv LANG LC_* COLORTERM NO_COLOR
          ^~~~
/etc/ssh/sshd_config:117:44: syntax error
Subsystem  sftp  /usr/lib/openssh/sftp-server
                 ^
What now? ^C
visudo exiting due to signal: Interrupt
admin@dc-node-01:~/.ssh $ sudo systemctl restart ssh
```

**This is the single best-practice moment in the build, and it is worth understanding precisely.** Editing through `visudo -f` (rather than `nano` directly) means the file is edited in a **temporary copy** — hence `sshd_config.tmp` in Fig 29 — and **validated before it replaces the live file**. The validator rejected the buffer at line 117 and offered *"What now?"*. `^C` aborted, and **the live `sshd_config` was never modified with a broken directive**.

Why that matters here more than usual: a malformed `sshd_config` prevents `sshd` from starting. On a headless node reachable only over the network, an `sshd` that will not start is an unrecoverable lockout without physical access. The validate-before-replace pattern is what stands between a typo and a trip to wherever the Pi is plugged in.

### 4d. Hardening audit — the objective was missed — **Fig 45**

The manual's §4c requires three directives. The edit session of Fig 29 was **aborted** at the syntax error (Fig 31), so `systemctl restart ssh` reloaded a configuration whose hardened state was unproven. It was then audited directly:

```bash
admin@dc-node-01:~ $ sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication'
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication yes
```

**✗ Verify FAILED — two of three directives are not in the required state.**

| Directive | Required | Actual | Verdict |
|---|---|---|---|
| `PubkeyAuthentication` | `yes` | `yes` | ✅ |
| `PasswordAuthentication` | `no` | **`yes`** | ❌ password login still accepted |
| `PermitRootLogin` | `no` | **`without-password`** | ❌ root may still log in by key |

This is the **effective, running configuration** — `sshd -T` prints what the daemon actually resolved after parsing `sshd_config` and every drop-in under `/etc/ssh/sshd_config.d/`, so there is no ambiguity and no need to reason about which file won.

Two things worth naming:

- **`permitrootlogin without-password`** is Debian's shipped default, not a value anyone typed. It is often written `prohibit-password`; it permits root to authenticate **by key** while refusing root passwords. Safer than `yes`, but it is **not** the `no` the manual requires — the node still has a reachable root login path.
- **Fig 30 proved the key works; it never proved passwords were refused.** Pubkey and password authentication are not mutually exclusive, and this audit is what separates the two claims. A green checkmark on "key login works" was hiding an open credential-guessing vector — which is precisely why the manual asks for a config read-back rather than a successful login as evidence.

**The remediation is §14.2.** Logged as **D3 — confirmed open**, upgraded from *unverified* to *failed* on the strength of Fig 45.

> **Fig 30** — key-based login from PowerShell

```
PS C:\Users\jredj\.ssh> ssh admin@172.20.10.10
Linux dc-node-01 6.18.34+rpt-rpi-2712 … aarch64
Last login: Wed Jul 22 15:17:15 2026 from 172.20.10.3
admin@dc-node-01:~ $
```

**✓ Verify (partial):** shell obtained with **no password prompt** — public-key authentication is working end to end, and this was confirmed in a **separate session while the original remained open**, which is the correct and non-negotiable sequencing. See D3 above for what remains unproven.

---

## 5. Host firewall — `ufw` — **Figs 32 → 33**

```bash
sudo apt install ufw -y
```
> **Fig 32** — pulls `ufw` plus `iptables`, `libip4tc2`, `libip6tc2`; 562 kB download, 9.7 MB installed

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

> **Fig 33** — the result

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
```

**✓ Verify PASSED.** Default-deny inbound, allow outbound, SSH explicitly permitted.

Four details worth reading rather than skimming:

- **Order is the safety property.** `ufw allow OpenSSH` was issued **before** `ufw enable`. Reversed, `enable` would have applied default-deny with no SSH exception and cut the session instantly — the same class of self-inflicted network lockout as §3.3, and the reason the manual sequences it this way.
- **Both v4 and v6 rules were created.** This node holds **global IPv6 addresses** (Figs 3, 21) — a v4-only rule would leave the node reachable over IPv6 while appearing firewalled. `ufw` handling both by default is exactly right, and confirming it in the output is the check most people skip.
- **`OpenSSH` is an application profile**, not a raw port number. It resolves via `/etc/ufw/applications.d` and keeps the intent legible: the rule says *why* the port is open, not merely *that* it is.
- **`disabled (routed)`** confirms the node is not forwarding between interfaces — correct for an endpoint, and something to revisit deliberately if it ever hosts container networks that need it.

---

## 6. Brute-force protection — `fail2ban` — **Figs 34 → 35**

```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

> **Fig 34** — installs `fail2ban` with `python3-systemd`, `python3-pyinotify`, `python3-pyasyncore`, `whois`
> **Fig 35** — jail status

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=ssh.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

**✓ Verify PASSED** — the `sshd` jail exists and is active.

- **`Journal matches: _SYSTEMD_UNIT=ssh.service`** shows the jail reading the **systemd journal** rather than tailing `/var/log/auth.log`. This is the modern backend and it matters: a file-based filter on a journal-only system watches a file that never fills, producing a jail that looks healthy and bans nothing.
- **`jail.conf` → `jail.local` is the correct customisation pattern.** Package upgrades overwrite `jail.conf`; `jail.local` overrides it and survives. Editing the wrong one means your tuning disappears at the next `apt upgrade` — silently.
- The `python3-systemd` dependency is the concrete reason the journal backend works at all.
- All counters at zero is the expected state for a node with no exposure yet — it is a **baseline**, not proof of efficacy. Efficacy would need a deliberate failed-login test:
  ```bash
  # from the workstation, several times, then:
  sudo fail2ban-client status sshd     # Currently failed should climb
  ```

---

## 7. Automatic security updates — **Figs 36 → 37**

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

> **Fig 36** — installs `unattended-upgrades` + `python3-distro-info`
> **Fig 37** — the important lines

```
Creating config file /etc/apt/apt.conf.d/20auto-upgrades with new version
Creating config file /etc/apt/apt.conf.d/50unattended-upgrades with new version
Created symlink '/etc/systemd/system/multi-user.target.wants/unattended-upgrades.service'
  → '/usr/lib/systemd/system/unattended-upgrades.service'
```

**✓ Verify PASSED.** The manual's check is that `/etc/apt/apt.conf.d/20auto-upgrades` exists — Fig 37 shows it being **created**, and the systemd symlink into `multi-user.target.wants` confirms the service is **enabled at boot**, not merely installed.

The two files divide cleanly: `20auto-upgrades` sets *how often* (`Update-Package-Lists`, `Unattended-Upgrade` intervals); `50unattended-upgrades` sets *what qualifies* (origins/suites — security only by default). Recommended read-back for Appendix B:
```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
sudo unattended-upgrades --dry-run --debug
```

This is the control that answers the 60-pending-packages observation of Fig 5. Note also `Not Upgrading: 128` recurring across Figs 32/34/36 — a standing backlog, and precisely what an unattended policy exists to keep from growing.

---

## 8. Time synchronisation — **Fig 38**

```bash
timedatectl list-timezones          # long list, America/… visible
timedatectl set-timezone America/Toronto
timedatectl
```

```
               Local time: Wed 2026-07-22 15:30:46 EDT
           Universal time: Wed 2026-07-22 19:30:46 UTC
                 RTC time: Wed 2026-07-22 19:30:46
                Time zone: America/Toronto (EDT, -0400)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

**✓ Verify PASSED** — `System clock synchronized: yes` and `NTP service: active`.

- The **`-0400` EDT offset** is correct for July and shows the tzdata rules are applied, not just a fixed offset.
- **`RTC in local TZ: no`** is the correct posture: the hardware clock runs UTC and local time is derived. An RTC in local time breaks across DST transitions and is a perennial dual-boot artifact.
- Local and Universal differ by exactly 4 hours with the RTC matching UTC — internally consistent.
- Note this is a *re-application* after the rebuild: the Imager set the timezone (Fig 0.5), the reflash reset it, and it was set again explicitly. Another instance of "verify on the target, not on the tool."

Correct time is load-bearing for everything downstream: `fail2ban` correlates journal timestamps, TLS certificates validate against wall-clock, and any TOTP later fails outright on a skewed clock.

---

## 9. Core tooling — **Fig 39**

```bash
sudo apt install -y git vim htop curl wget net-tools dnsutils \
  build-essential ca-certificates gnupg tmux tree
```

```
Note, selecting 'bind9-dnsutils' instead of 'dnsutils'
git is already the newest version (1:2.47.3-0+deb13u1).
htop is already the newest version (3.4.1-5).
wget is already the newest version (1.25.0-2).
net-tools is already the newest version (2.10-1.3).
Upgrading: curl  libcurl3t64-gnutls  libcurl4t64
Installing: bind9-dnsutils  vim
```

**✓ Verify PASSED** — `git` resolves at 2.47.3, and the rest are present or newly installed.

- **`dnsutils` → `bind9-dnsutils`** is a **virtual-package resolution**, not an error. `dnsutils` is now a transitional name; APT reported the substitution rather than performing it silently. This is the package manager being honest about a rename, and reading the `Note:` line is how you catch the cases where it *isn't* what you wanted.
- `+deb13u1` in the git version string is a Debian 13 security/stability revision — corroborating Trixie once more.
- Most tools shipping preinstalled reflects Raspberry Pi OS being a desktop-derived image; on a minimal server image this step installs far more.

---

## 10. Portfolio workflow — Git and GitHub — **Figs 40 → 42**

```bash
git config --global user.name  "Redji Jean Baptiste"
git config --global user.email "…"
ssh-keygen -t ed25519 -C "dc-node-01"
ssh -T git@github.com
```

> **Fig 40** — key generation, and the first attempt failing

```
Your identification has been saved in /home/admin/.ssh/id_ed25519
Your public key has been saved in /home/admin/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:hajSGndS9y3mbRVTPg94YdjObmepV4VIKJPAp2gwbTs dc-node-01

$ ssh -T git@github.com
The authenticity of host 'github.com (64:ff9b::8c52:7104)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
Are you sure you want to continue connecting? yes
git@github.com: Permission denied (publickey).
```

**`Permission denied (publickey)` here is a correct, expected result, not a failure.** The keypair exists on the node; GitHub has never seen the public half. The key must be pasted into **GitHub → Settings → SSH and GPG keys** before authentication can succeed. Capturing the denial *and then* the success is stronger evidence than showing only the success — it demonstrates the mechanism, not just the outcome.

Two details:

- **This is a second, distinct keypair**, commented `dc-node-01`, separate from the workstation's `jredj@MSI` key of Fig 28. Correct practice: each machine holds its own private key and each is independently revocable. The comment is what makes revocation possible without guesswork later.
- **`github.com (64:ff9b::8c52:7104)`** is an IPv6 address in the **NAT64 well-known prefix** `64:ff9b::/96` — the node reached GitHub through a NAT64/DNS64 translation path. Consistent with the IPv6-heavy environment throughout, and the sort of detail worth noticing before it becomes a confusing outage.
- GitHub's own fingerprint `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU` is published by GitHub and should be compared against their documentation rather than accepted on sight.

> **Fig 41** — after adding the public key to GitHub

```
$ ssh -T git@github.com
Hi RedjiJB! You've successfully authenticated, but GitHub does not provide shell access.
```

**✓ Verify PASSED** — GitHub greets the account **`RedjiJB` by name**, confirming the key is bound to the correct identity. *"GitHub does not provide shell access"* is the expected terminus: `-T` disables PTY allocation and GitHub serves Git over SSH, not a login shell.

> **Fig 42** — repository initialisation

```
Initial commit
nothing to commit (create/copy files and use "git add" to track)
admin@dc-node-01:~/cst-portfolio $
```

The repository is initialised at `~/cst-portfolio` with the working tree still empty. **The portfolio skeleton and first real commit remain outstanding:**

```bash
cd ~/cst-portfolio
mkdir -p L1-CST8207-linux/p00-node-foundation/{artifacts,evidence}
# place this document at .../p00-node-foundation/README.md
git add . && git commit -m "P00: node foundation baseline"
git remote add origin git@github.com:RedjiJB/cst-portfolio.git
git push -u origin main
```

---

## 11. Container substrate — Docker — **Figs 42 → 44**

```bash
curl -fsSL https://get.docker.com | sh
```

> **Fig 42** (lower) — the install script's actions, echoed

```
# Executing docker install script, commit: 5ce20f2eef3615d08fea941eda5a109e949e8ebf
+ sh -c install -m 0755 -d /etc/apt/keyrings
+ sh -c curl -fsSL "https://download.docker.com/linux/debian/gpg" -o /etc/apt/keyrings/docker.asc
+ sh -c echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable" > /etc/apt/sources.list.d/docker.list
+ sh -c apt-get -y -qq install docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-ce-rootless-extras docker-buildx-plugin docker-model-plugin
+ sh -c systemctl enable --now docker.service
INFO: Docker daemon enabled and started
Client: Docker Engine - Community  Version: 29.6.2   API version: 1.55
```

Reading the echoed commands is worthwhile — piping a remote script to a shell is a real supply-chain exposure, and the script prints every privileged action it takes. Visible here: a **GPG key installed to `/etc/apt/keyrings`** and the repository line **`signed-by=`** that key, so subsequent packages are cryptographically verified through APT's normal trust path. `arch=arm64` and suite `trixie` both corroborate the platform. Docker Engine **29.6.2** is installed and the daemon **enabled at boot**.

> **Fig 43** — `sudo docker run --rm hello-world`

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
Digest: sha256:c3cbe1cc1aa588a64951ac6286e0df7b27fe2e6324b1001c619bb358770c0178
Hello from Docker!
This message shows that your installation appears to be working correctly.
  2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
     (arm64v8)
```

The daemon pulled the **`arm64v8`** variant — multi-architecture image resolution working correctly for this platform, and the practical vindication of choosing the 64-bit OS in §0.3.

> **Fig 44** — `docker compose version` → `Docker Compose version v5.3.1`

**✓ Verify PARTIAL.**

- ✅ `hello-world` runs; the daemon is functional end to end (registry pull → image → container → stdout).
- ✅ `docker compose` resolves at v5.3.1, and notably **without `sudo`** — the CLI plugin is on the user's path.
- ❌ **`hello-world` required `sudo`.** The manual's verification is explicit: *"`hello-world` runs without sudo."* The `usermod -aG docker $USER` step and the re-login that activates it were not completed. Logged as **D4**:
  ```bash
  sudo usermod -aG docker $USER
  newgrp docker            # or log out and back in
  docker run --rm hello-world      # must now succeed WITHOUT sudo
  ```

> **Security note, not a formality.** Membership in the `docker` group is **equivalent to root**: any member can start a container that bind-mounts `/` and edit anything on the host. This is not a hardening regression to be embarrassed about — it is an accepted, well-documented trade-off of running Docker at all — but it belongs in §9 explicitly rather than being smuggled in as a convenience fix. The `docker-ce-rootless-extras` package installed in Fig 42 offers the alternative if that trade-off is ever declined.

---

## 12. Final verification checklist

| # | Item | Evidence | Status |
|---|---|---|---|
| 1 | Booting from SSD | Fig 4, Fig 24 — `/dev/mmcblk0p2` | ❌ **microSD** (§8.2, accepted) |
| 2 | Hostname correct | Figs 1, 8, 15 | ✅ |
| 3 | Timezone + NTP | Fig 38 | ✅ |
| 4 | Static IP | Fig 21 | ⚠️ Set, then invalidated by relocation (§3.6) |
| 5 | Key-based SSH works | Fig 30 | ✅ |
| 6 | Password + root login disabled | Figs 45 → 57 → 58 → **59** | ✅ **Closed** — negative test refuses password auth (§15.1) |
| 7 | `ufw` active, default-deny | Fig 33 | ✅ |
| 8 | `fail2ban` sshd jail active | Fig 35 | ✅ |
| 9 | `unattended-upgrades` enabled | Fig 37 | ✅ |
| 10 | Core tooling present | Fig 39 | ✅ |
| 11 | Docker runs **without** sudo | Fig 43 | ❌ **D4** |
| 12 | `docker compose` present | Fig 44 | ✅ |
| 13 | GitHub SSH auth | Fig 41 | ✅ |
| 14 | Portfolio repo committed + pushed | Fig 42 | ⬜ Initialised, empty |
| 15 | Everything survives `sudo reboot` | — | ⬜ **Not yet performed** |

**Item 15 is the one that certifies the rest.** Every control above is claimed on the basis of its live state; a reboot is what distinguishes *configured* from *persistent*. Given that §3.5 already showed a profile silently renamed across a rebuild, this is not a formality:

```bash
sudo reboot
# then, after it comes back:
hostnamectl; ip a; timedatectl
sudo ufw status verbose
sudo fail2ban-client status sshd
sudo systemctl is-enabled ssh docker fail2ban unattended-upgrades
docker run --rm hello-world
```

---

## 13. Troubleshooting log

### Faults encountered and resolved

| # | Symptom | Root cause | Resolution |
|---|---|---|---|
| T1 | Imager greyed out **NEXT** — *"Password is too short (min 8 characters)"* (Fig 0.6) | WPA2-PSK requires an 8–63 char PSK; the Imager validates pre-write | Re-entered full passphrase. **Resolved** |
| T2 | Windows AutoPlay: *"There's a problem with this drive"* (Fig 0.3) | Windows cannot read `ext4` and misreports the disk | Dismissed. **Never** run the Windows repair tool on a fresh image. **Resolved by ignoring** |
| T3 | `df -h` shows `mmcblk0p2`, not an SSD (Figs 4, 24) | The "USB Mass Storage Device" was a card reader holding the microSD | Continued on microSD; endurance risk carried to §9. **Accepted deviation** |
| T4 | Manual's `nmcli` targets `"Wired connection 1"` — no such connection (Fig 10) | Real profile was `netplan-wlan0-Calypso`; node is on Wi-Fi | Discover with `nmcli connection show` before modifying. **Method corrected** |
| T5 | Netplan could re-render and revert `nmcli` changes | Netplan rendering above NetworkManager | Post-rebuild the profile is plain `Calypso` (Fig 19) — Netplan no longer owns it. **Resolved by rebuild** |
| T6 | `Error: unknown connection 'netplan-wlan0-Calypso\n'` (Fig 11) | Unterminated `"`; shell continuation `>` absorbed a newline into the argument | Retyped with balanced quotes. **Resolved** |
| **T7** | **`Network error: Software caused connection abort` — node lost** (Figs 12→13) | **Applied the manual's example `192.168.1.50/24` to a node on `10.0.0.0/24`; `nmcli` accepted it silently** | **Re-imaged the microSD (Fig 0.12); new Machine ID proves rebuild (Fig 15). Re-applied with real values `10.0.0.2/24`. Resolved** |
| T8 | Workstation showing hundreds of `169.254.x.x` (Fig 14) | APIPA link-local — no DHCP server answering on that segment | Diagnostic only; distinguished a segment fault from a node fault. **Understood** |
| T9 | `-bash: supd: command not found` (Fig 16) | Typo in the right side of an `&&` chain | Left side (`apt update`) had already succeeded; re-ran the upgrade. **Harmless** |
| T10 | `Error: argument 'conneccction' not understood` (Fig 20) | Typo in the `nmcli` subcommand | Retyped. **Failed loudly — the safe kind** |
| T11 | `passwd: You may not view or modify password information for redji` (Fig 25) | Unprivileged users may change only their own password; `/etc/shadow` is root-only | `sudo passwd redji`. **Correct refusal, not a fault** |
| T12 | `/etc/ssh/sshd_config:117:44: syntax error` (Fig 31) | Malformed directive in the edit buffer | Caught by validate-before-replace; `^C` aborted, live file untouched. **Prevented an unrecoverable lockout** |
| T13 | `git@github.com: Permission denied (publickey)` (Fig 40) | Public key not yet registered with GitHub | Added key in GitHub settings → `Hi RedjiJB!` (Fig 41). **Resolved** |
| T14 | Static `10.0.0.2` became inert after relocation (Figs 22–23) | A static address is bound to one subnet; the node moved across three networks | Move to a **router-side DHCP reservation** keyed to MAC. **Design change recorded** |

### Open defects

| # | Defect | Impact | Fix |
|---|---|---|---|
| **D1** | DNS secondary set to `8.4.4.4` — typo for `8.8.4.4` (Fig 20) | Invisible in normal use; fails exactly when the primary is down | §14.3 |
| **D2** | Pre-existing account `redji` was granted `sudo` and a new password (Fig 25) | `redji` was **not** created in this build — `adduser` reported it already existed, and `testuser1` was the account deliberately created for the user-management requirement (Fig 26). An account that predates the current install now holds full admin rights on a least-privilege node. | §14.4 — audit, then lock or remove |
| **D3** | **CONFIRMED:** `passwordauthentication yes`, `permitrootlogin without-password` (Fig 45) | Password login is **accepted**; root retains a key-based login path. The primary credential-guessing vector the manual exists to close is still open. | §14.2 — **highest priority** |
| **D4** | `docker` still requires `sudo` (Fig 43) | Manual's verification unmet | §14.5 — note the root-equivalence trade-off (§9) |
| **D5** | `~/.ssh` and `authorized_keys` permissions unverified (Fig 28) | Over-permissive modes cause `sshd` to ignore the file silently — and would cause a **lockout** the moment D3 is fixed | §14.1 — **must precede §14.2** |
| **D6** | Lab account `testuser1` retained (Fig 26) | Intentional — this is the account created to satisfy the user-management requirement, and its `adduser` transcript is the evidence for it. It carries a password and no `sudo`. Keep for assessment; remove before the node enters service. | §14.6 |
| **D7** | Reboot-survival verification not performed | *Configured* ≠ *persistent* | §14.7 |

---

## 14. Remaining work — runbook to close the lab

> **Run these in order.** §14.1 must precede §14.2: disabling password authentication while `~/.ssh` permissions are wrong produces an immediate, unrecoverable lockout on a node with no attached console. This is the same sequencing principle that the firewall step (§5) and the manual's §4 both depend on — open the new door before closing the old one.
>
> **Keep your current SSH session open through §14.2.** Every test opens a *second* session. If a test fails, you still have the first one to repair from.

### 14.1 Fix key permissions first — closes D5

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
ls -la ~/.ssh
```

**✓ Verify:** `drwx------` on `.ssh` and `-rw-------` on `authorized_keys`. If either is group- or world-writable, `sshd` **silently ignores the file** — the key stops working with nothing useful in the client output.

Confirm the key is actually being accepted before you remove the password fallback:

```bash
sudo journalctl -u ssh --since "-10min" | grep -i "Accepted publickey"
```

> **Evidence to capture → Fig 46.**

### 14.2 Harden the SSH daemon — closes D3 *(highest priority)*

Do **not** edit `/etc/ssh/sshd_config` directly. Fig 31 showed that file already carries a syntax error at line 117, and package upgrades overwrite it. Use a drop-in instead — it takes precedence, survives upgrades, and keeps your intent in one readable file:

```bash
sudo tee /etc/ssh/sshd_config.d/99-dc-hardening.conf >/dev/null <<'EOF'
# DC-FOUNDATION-01 — P00 host hardening
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
EOF
```

`KbdInteractiveAuthentication no` is not optional padding: without it, PAM's keyboard-interactive path can still present a password prompt even with `PasswordAuthentication no`, leaving the vector half-open.

**Validate before restarting** — this is the habit Fig 31 already proved valuable:

```bash
sudo sshd -t && echo "CONFIG OK"
```

If that prints anything other than `CONFIG OK`, **stop and fix it** — do not restart the daemon.

```bash
sudo systemctl restart ssh
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractive'
```

**✓ Verify — expected output** (compare directly against Fig 45):

```
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

**Then the negative test**, from PowerShell on the workstation — this is the evidence that actually closes D3, because it proves what is *refused*, not what is permitted:

```powershell
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no admin@<node-ip>
```

**✓ Verify:** must fail with `Permission denied (publickey).` A password prompt appearing here means the hardening did **not** take effect — revert the drop-in from your still-open session and re-diagnose.

Finally, confirm normal key login still works in a fresh session:

```powershell
ssh admin@<node-ip>
```

> **Evidence to capture → Fig 47** (`sshd -T` after), **Fig 48** (the negative test failing as required).

### 14.3 Correct the DNS secondary — closes D1

The connection name changed once already (Figs 10 → 19), so **discover it again rather than assuming `Calypso`**:

```bash
nmcli connection show
sudo nmcli connection modify "<connection-name>" ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli connection up "<connection-name>"
resolvectl status | grep -A2 "DNS Servers"
```

**✓ Verify:** both `8.8.8.8` and `8.8.4.4` listed — no `8.4.4.4`. Confirm resolution end to end:

```bash
dig +short github.com
```

### 14.4 Audit the `redji` account — closes D2

```bash
getent passwd redji          # UID, home directory, login shell
sudo lastlog -u redji        # has it ever logged in?
groups redji                 # confirm what sudo grant took effect
sudo ls -la /home/redji 2>/dev/null
```

A UID in the 1000s with a real home directory means a normally-created interactive account; the question is who created it and when. Since `testuser1` (Fig 26) is the account you deliberately created for the user-management requirement, `redji` serves no purpose in this build. Remove the privilege and disable the login:

```bash
sudo gpasswd -d redji sudo   # revoke admin rights
sudo passwd -l redji         # lock the password
```

Or remove it outright if the audit shows it was never used:

```bash
sudo deluser --remove-home redji
```

**✓ Verify:** `groups redji` no longer lists `sudo`; `sudo passwd -S redji` shows `L` (locked).

### 14.5 Docker without sudo — closes D4

```bash
sudo usermod -aG docker $USER
newgrp docker
docker run --rm hello-world
docker compose version
```

**✓ Verify:** `hello-world` succeeds with **no `sudo`**, satisfying the manual's §11 verification. `newgrp` applies the group to the current shell; a full logout also works and is what makes it stick for future sessions.

> **State this as a decision, not a convenience.** `docker` group membership is **root-equivalent** — any member can run a container that bind-mounts `/` and rewrite anything on the host. That is an accepted trade-off of running Docker, documented in §9, not an oversight. `docker-ce-rootless-extras` is already installed (Fig 42) if you ever decline the trade-off.

> **Evidence to capture → Fig 49.**

### 14.6 Account hygiene — closes D6

`testuser1` is the account you created to satisfy the user-management requirement, and Fig 26 is its evidence — **keep it until the lab is assessed.** Confirm it holds no privilege it shouldn't:

```bash
groups testuser1             # must NOT include sudo
sudo passwd -S testuser1
```

Then, before the node enters service:

```bash
sudo deluser --remove-home testuser1
```

### 14.7 Reboot-survival verification — closes D7

**This is the step that certifies every other one.** Each control above is currently claimed on its live state; a reboot is what distinguishes *configured* from *persistent* — and §3.5 already showed a connection profile silently renamed across a rebuild, so this is not a formality here.

```bash
sudo reboot
```

Then, once it returns:

```bash
hostnamectl                                    # dc-node-01, Trixie
ip a                                           # addressing as expected
timedatectl                                    # synchronized: yes
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication'
sudo ufw status verbose                        # active, default deny (incoming)
sudo fail2ban-client status sshd               # jail active
sudo systemctl is-enabled ssh docker fail2ban unattended-upgrades
docker run --rm hello-world                    # still no sudo
```

**✓ Verify:** every line matches its pre-reboot state, and all four units report `enabled`.

> **Evidence to capture → Fig 50** (the full post-reboot block). This is the single most valuable screenshot remaining — it converts fourteen individual claims into one demonstrated property.

### 14.8 Commit and push the portfolio

```bash
cd ~/cst-portfolio
mkdir -p L1-CST8207-linux/p00-node-foundation/{artifacts,evidence}
# place this document at .../p00-node-foundation/README.md
# place renamed screenshots in .../evidence/  (see the file-mapping table)

# Appendix A + B artifacts:
cd L1-CST8207-linux/p00-node-foundation
history > artifacts/setup-history.txt
sudo sshd -T > artifacts/sshd-effective.txt
sudo ufw status verbose > artifacts/ufw-status.txt
sudo fail2ban-client status sshd > artifacts/fail2ban-status.txt
nmcli connection show > artifacts/nmcli-connections.txt
cat /etc/apt/apt.conf.d/20auto-upgrades > artifacts/auto-upgrades.txt

cd ~/cst-portfolio
git add .
git commit -m "P00: node foundation baseline — hardened, verified, reboot-persistent"
git remote add origin git@github.com:RedjiJB/cst-portfolio.git
git push -u origin main
```

> **⚠ Before pushing.** Review `evidence/` for anything personal. The capture set for this build included an unrelated online-banking screenshot showing account balances and transaction history. **A public repository is permanent** — check the folder, not your memory of it.

Also confirm no secrets rode along in the artifacts:

```bash
grep -ri "password\|passphrase\|BEGIN OPENSSH PRIVATE KEY" artifacts/ || echo "clean"
```

### 14.9 Optional — the deferred `raspi-config` items

Not required by the manual's checklist, but they close §2's remaining rows:

```bash
sudo raspi-config
# Performance Options → GPU Memory → 16   (headless server needs no GPU RAM)
# Advanced Options → Boot Order           (only relevant if migrating to SSD, §8.2)
```

---

## 15. Remediation results

*Executed 2026-07-22 against the runbook in §14. Figures 45–60.*

### 15.1 The include-ordering trap — D3, root-caused and closed

**Fig 46** confirmed permissions (`drwx------` on `~/.ssh`, `-rw-------` on `authorized_keys` and `id_ed25519`) — **D5 closed**. **Fig 47** wrote the drop-in and `sshd -t` returned `CONFIG OK`.

**Fig 48** — after `systemctl restart ssh`, the audit still failed, but only partly:

```
permitrootlogin no                  ← our file won
pubkeyauthentication yes
kbdinteractiveauthentication no     ← our file won
passwordauthentication yes          ← something overrode us
```

Two of four directives applied. That asymmetry is the whole diagnosis: the file **was** being read, so the problem was not the file — it was a conflict on one directive.

**Fig 57** — the search that found it:

```
/etc/ssh/sshd_config.d/50-cloud-init.conf:1:PasswordAuthentication yes
/etc/ssh/sshd_config.d/99-dc-hardening.conf:2:PasswordAuthentication no
```

**`sshd_config` resolves the FIRST value obtained, not the last.** This is the reverse of nearly every other configuration system — `.bashrc`, `sysctl.d`, systemd drop-ins, CSS all let later declarations win. Debian places `Include /etc/ssh/sshd_config.d/*.conf` at the *top* of the main file, and within that directory files load in lexical order. `50-cloud-init.conf` was therefore read first and its `yes` became final; the `no` in `99-` was read afterwards and discarded.

The file is written by **cloud-init**, which re-asserts password authentication as part of its own provisioning. `PermitRootLogin` and `KbdInteractiveAuthentication` applied precisely because cloud-init does not set them — nothing was competing.

**Fig 58** — the fix is one rename, not an edit:

```bash
sudo mv /etc/ssh/sshd_config.d/99-dc-hardening.conf \
        /etc/ssh/sshd_config.d/00-dc-hardening.conf
sudo sshd -t && sudo systemctl restart ssh
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractive'
```

```
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
```

Sorting to `00-` puts our file ahead of every other drop-in, so its values are the first obtained and therefore the ones that stand. Note what was *not* done: `50-cloud-init.conf` was left intact. Deleting it would work until cloud-init regenerated it; winning on ordering is durable and leaves the other tool's file untouched.

**Fig 59 — the negative test, from the workstation:**

```
PS C:\> ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no admin@192.168.1.1
admin@192.168.1.1: Permission denied (publickey).
```

**✓ D3 CLOSED.** The server refused password authentication and named public-key as the only method it accepts. This is the evidence the whole §4 objective rests on — and it is a *refusal*, which no amount of successful logins could ever have demonstrated.

> **Why this is the most valuable finding in the report.** Three separate signals said the hardening was fine: the drop-in existed with correct contents, `sshd -t` validated it, and the daemon restarted without error. All three were true and the control was still not in effect. Only reading the daemon's *resolved* configuration (`sshd -T`), and then testing the refusal, exposed the gap. Configuration that is *present* is not configuration that is *effective*.

### 15.2 Docker, accounts, and reboot persistence

| Item | Result | Evidence |
|---|---|---|
| `docker run --rm hello-world` **without `sudo`** | ✅ **D4 closed** | Figs 50–51 |
| `testuser1` held **`sudo`** — unintended | 🔴 found by audit | Fig 52 |
| `sudo gpasswd -d testuser1 sudo` → `testuser1 : testuser1 users` | ✅ privilege revoked | Fig 60 |
| `redji` no longer exists (`getent` returned nothing) | ✅ **D2 dissolved** — see §15.4 | Fig 49 |
| `ufw` active, default-deny, 22/tcp v4+v6 **after reboot** | ✅ persistent | Figs 53–54 |
| `fail2ban` sshd jail active **after reboot** | ✅ persistent | Fig 54 |
| `ssh docker fail2ban unattended-upgrades` all `enabled` | ✅ persistent | Fig 54 |
| `permitrootlogin no` **after reboot** | ✅ persistent | Fig 53 |
| Clock `synchronized: yes` | ❌ **regressed to `no`** — **D8** | Figs 53, 60 |

**Fig 52 is a finding produced by a check expected to pass.** `groups testuser1` returned `testuser1 : testuser1 sudo users` — the account created purely to demonstrate user management had been granted full administrative rights, with a usable password (`passwd -S` → `P`). On a node whose stated premise is least privilege, a demo account with `sudo` is the same defect class as D2, arrived at by a different route. Revoked in Fig 60.

**The reboot test earned its place.** It confirmed six controls persisted — and caught one regression that a live-state check would have reported as healthy (§15.3). Verifying persistence is not ceremony; it found something.

### 15.3 D8 (new) — clock synchronisation regressed after reboot

```
     RTC time: Thu 1970-01-01 02:29:21
System clock synchronized: no
  NTP service: active
```

Fig 38 recorded `synchronized: yes` before the reboot; Figs 53 and 60 record `no` after it, and a `systemd-timesyncd` restart did not recover it.

**The RTC at epoch is expected, not the defect.** The Pi 5 has a hardware RTC but ships **without the coin cell** that would back it, so it loses time at power-off; the OS compensates with `fake-hwclock` and NTP catch-up. Wall-clock time is in fact correct (`18:43:35 EDT`, matching `22:43:35 UTC`), so time was restored from somewhere — but `NTPSynchronized: no` means **no NTP source has been confirmed**, and the clock is therefore unverified rather than merely late.

**Probable cause — and it is a chain, not a coincidence.** The DNS remediation of §15.4 left the connection in a failed activation state. `systemd-timesyncd` must resolve `*.debian.pool.ntp.org` before it can reach any server; with resolution broken it fails **silently**, reporting `NTP service: active` while synchronising with nothing. `active` describes the daemon, not the outcome — the same *present-vs-effective* distinction as D3, in a different subsystem.

**Fix the network first, then re-check.** Diagnosis order matters here:

```bash
getent hosts github.com                       # DNS working?
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd
journalctl -u systemd-timesyncd -n 20 --no-pager
timedatectl show -p NTPSynchronized --value   # target: yes
```

**Why it matters beyond a checkbox:** `fail2ban` correlates journal timestamps to decide what constitutes a burst of failures; TLS validates certificate windows against wall-clock; any future TOTP fails outright on skew. An unverified clock quietly weakens three controls that all report themselves as healthy.

### 15.4 D1 — DNS correction attempted, network activation failed

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.dns "8.8.8.8 8.8.4.4" \
  && sudo nmcli connection up "Wired connection 1"
```

```
Error: Connection activation failed: IP configuration could not be reserved
       (no available address, timeout, etc.)
```

**`nmcli connection up` is not a reload — it deactivates the connection and brings it back from scratch**, which on a DHCP profile means a fresh lease request. The request timed out, leaving the connection down. The `modify` had already been written to the profile; only the activation failed. **D1 remains open** pending re-activation and verification:

```bash
nmcli -f NAME,DEVICE,STATE connection show
sudo nmcli connection up "Wired connection 1"
nmcli -g ipv4.dns connection show "Wired connection 1"
cat /etc/resolv.conf
```

Two observations worth carrying forward:

- **This is the third distinct connection-profile name in one build** — `netplan-wlan0-Calypso` (Fig 10) → `Calypso` (Fig 19) → `"Wired connection 1"` (Fig 60), the last being the manual's original example name, now genuinely present. The "discover, never assume" rule of §3.1 has now been validated three times.
- **Re-activating the connection you are connected over is the same hazard as §3.3 and §5**, in a third tool. The session survived only because SSH ran over an interface that stayed up. On a single-homed node this command can disconnect you outright — and unlike the §3.3 case, it fails *loudly*, which is what made it recoverable.

### 15.5 Portfolio push — branch mismatch

```
[master (root-commit) fe1763c] P00: node foundation baseline - hardened, verified, reboot-persistent
 6 files changed, 266 insertions(+)
$ git push -u origin main
error: src refspec main does not match any
error: failed to push some refs to 'github.com:RedjiJB/cst-portfolio.git'
```

`git init` created **`master`**; the push targeted **`main`**. Git does not rename or infer — `main` simply did not exist. The commit itself is sound, and the pre-push secret sweep returned **`clean`**.

```bash
git branch -M main
git push -u origin main
```

**Still outstanding:** the 6 committed files are all `artifacts/`. **`README.md` (this document) and the `evidence/` screenshots are not yet in the repository**, so the commit is not yet the deliverable.

### 15.6 Defect status after remediation

| # | Defect | Status |
|---|---|---|
| D1 | DNS secondary `8.4.4.4` | ✅ **Closed** — corrected and resolving (§15.7) |
| D2 | `redji` holding `sudo` | ✅ Dissolved — account absent on current install |
| D3 | Password auth / root login enabled | ✅ **Closed** — root-caused to include ordering; negative test passed (Fig 59) |
| D4 | `docker` required `sudo` | ✅ Closed (Fig 51) |
| D5 | `~/.ssh` permissions unverified | ✅ Closed (Fig 46) |
| D6 | `testuser1` privileges | ✅ `sudo` revoked (Fig 60); remove account before service |
| D7 | Reboot survival unverified | ✅ Closed — 6 controls persisted; 1 regression found (Figs 53–54) |
| **D8** | **Clock not synchronised after reboot** | ✅ **Closed** — root cause was D1's stalled activation, not NTP (§15.7) |
| D9 | Portfolio push | 🟡 `git branch -M main`; README + evidence outstanding (§15.5) |

---

## 15.7 One stalled activation, three symptoms — D1 and D8 closed together

**Figs 61–62.**

`Wired connection 1` sat in state **`activating`** — DHCP never completed on `eth0`. NetworkManager therefore never installed working resolver configuration, and `/etc/resolv.conf` was left holding a **single IPv6 link-local nameserver**:

```
nameserver fe80::640c:91ff:fe95:ac64%wlan0
```

That one stalled activation produced three symptoms that looked unrelated:

| Symptom | Where it appeared | Actual cause |
|---|---|---|
| `getent hosts github.com` returns nothing | §15.4 | No usable resolver |
| `NTPSynchronized: no` with `NTP service: active` | §15.3 (D8) | timesyncd could not resolve `*.debian.pool.ntp.org` |
| `client_loop: send disconnect: Connection reset`; `Connection timed out` | Fig 61 | The wired link flapping under a failing activation |

**None of the three names the network as the fault.** A dead clock looks like an NTP problem; a dropped SSH session looks like a server problem; a failed lookup looks like a DNS problem. All three were one incomplete DHCP transaction. The diagnostic lesson is ordering: **fix the layer everything else stands on first**, then re-test upward — the two "separate" defects closed themselves.

The journal confirmed the diagnosis before the fix — `systemd-timesyncd` logged only:

```
Network configuration changed, trying to establish connection.
```

repeatedly, and never a server contact. **`NTP service: active` describes a running daemon, not a synchronised clock** — the same *present-vs-effective* distinction that D3 turned on (§15.1), in a second subsystem.

**Resolution — move the node onto the Wi-Fi profile with a static address:**

```bash
sudo nmcli connection modify "iPhone" \
  ipv4.addresses 172.20.10.10/28 \
  ipv4.gateway 172.20.10.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  ipv4.method manual
sudo nmcli connection up "iPhone"
```

```
Connection successfully activated (D-Bus active path: .../ActiveConnection/25)
```

`/28` is correct for an iOS hotspot — usable range `.2`–`.14`, so `.10` sits inside it. The wired profile was then stopped from competing for the default route:

```bash
sudo nmcli connection modify "Wired connection 1" connection.autoconnect no
```

**✓ Verify — both defects closed by the same change:**

```
admin@dc-node-01:~ $ getent hosts github.com
140.82.112.3    github.com

admin@dc-node-01:~ $ sudo systemctl restart systemd-timesyncd && sleep 20 \
                     && timedatectl show -p NTPSynchronized --value
yes
```

**D1 closed** — resolution working through the corrected `8.8.8.8 / 8.8.4.4` pair. **D8 closed** — `NTPSynchronized: yes`, restoring the property Fig 38 asserted and the reboot test disproved.

### A note on the self-SSH attempt

Fig 62 shows `ssh admin@172.20.10.10` issued **from the node itself**, returning `Permission denied (publickey)`. This is **not** a defect. `~/.ssh/authorized_keys` contains the workstation's key (`jredj@MSI`, Fig 28); the node's own keypair (`dc-node-01`, Fig 40) was generated for **GitHub** and was never added to its own authorized keys. The refusal is D3's hardening behaving correctly on a loopback connection — and it is a second, incidental confirmation that password fallback is gone.

### Management-path note

PuTTY stopped working after §15.1 with a public-key error. **Also not a defect:** PuTTY cannot read OpenSSH-format private keys and requires conversion to `.ppk` via PuTTYgen. The correct response is to convert the key or use the OpenSSH client — **never** to re-enable `PasswordAuthentication`, which would reopen D3. Management moved to PowerShell's OpenSSH client, which authenticates against the same Ed25519 key with no conversion.

**Addressing note.** The node is now reachable at a **static `172.20.10.10`** on the hotspot, which finally satisfies §3's objective in a stable form — though the §3.6 conclusion still stands: a node that changes networks needs a router-side reservation, not a host-side static, and a phone hotspot remains a bootstrap link rather than a permanent home.

---

# PART B — META REPORT

**Project ID:** P00
**Course mapping:** CST8207 GNU/Linux System Support · CST8182 Networking Fundamentals · CST8323 / CST8249 Network Security · CST8206 Foundation of IT Service Management
**Program/Level:** 1560X / 0156X, L1 foundation · **Tier:** T2
**Environment:** Raspberry Pi 5 Model B Rev 1.1 (8 GB) · Raspberry Pi OS 64-bit, Debian 13 "Trixie" · kernel 6.18.34+rpt-rpi-2712 · microSD boot, 58 G root · Docker Engine 29.6.2 · addressing varied across three networks during the build (§3.6)

## 1. Objective & scope

Provision a Raspberry Pi 5 from bare hardware into a hardened, reproducible server node that serves as the foundation for the DC-PI and CST project portfolios.

**In scope:** image preparation and first-boot customisation; baseline capture; system and firmware patching; base configuration; addressing; user and SSH key management; SSH daemon hardening; host firewall; intrusion protection; automatic patching; time synchronisation; core tooling; portfolio workflow; container substrate.

**Out of scope:** individual service deployments, which are the subject of subsequent projects and assume this node as their substrate.

## 2. Competencies demonstrated

- **Linux administration** — account lifecycle (`adduser`, `usermod -aG`, `passwd`), the `/etc/shadow` permission model, systemd unit enablement, package management including virtual-package resolution, filesystem inspection (CST8207)
- **Network configuration & diagnosis** — interface and carrier state, DHCP-vs-static lease semantics (`dynamic`/`valid_lft`/`noprefixroute`), NetworkManager and Netplan layering, APIPA fault identification, mDNS, multi-homed host diagnosis, NAT64 observation (CST8182)
- **Host hardening** — asymmetric key generation and distribution, host-key fingerprint verification, default-deny firewalling across both IP families, journal-backed brute-force mitigation, automated patching, validate-before-replace configuration editing (CST8323 / CST8249)
- **Service management & documentation** — reproducible build record, verification discipline, defect tracking, honest recording of a self-inflicted outage and its recovery (CST8206)
- **Resume bullet:** *"Provisioned and hardened a Linux server node from bare hardware — headless first-boot automation, Ed25519 key-based SSH, default-deny host firewall, fail2ban, automated patching, and a Docker substrate — recovering from a self-inflicted network lockout via a full documented rebuild, with every control verified against captured evidence."*
- **Job-target relevance:** baseline sysadmin/SecOps competency expected by DND/CSE and private-sector infrastructure roles.

## 3. Background

A node's security posture is set at provisioning; retrofitting hardening after services are exposed is error-prone, because every control added later must be reconciled against workloads already depending on the old behaviour. Establishing key-based SSH and default-deny firewalling *before any service exists* means there is no attack surface to regress.

Two provisioning details carry disproportionate weight. First, **Debian 12+ manages networking through NetworkManager**, not legacy `dhcpcd`, and on this image Netplan sits above NetworkManager as renderer — so the wrong tool fails *silently*. Second, **first-boot customisation** converts several later configuration steps into verification steps, which is what makes a build reproducible rather than merely repeatable. This build tested that property directly: when the node was lost, recovery cost one imaging pass.

## 4. Environment & tools

Raspberry Pi 5 Model B Rev 1.1, 8 GB RAM; microSD boot, 58 G root; Raspberry Pi OS 64-bit (Debian 13 Trixie), kernel 6.18.34+rpt-rpi-2712, `arm64`.

**Tooling:** Raspberry Pi Imager v2.0.10 · PuTTY · OpenSSH for Windows · `raspi-config` · `nmcli` / NetworkManager / Netplan · `rpi-eeprom-update` · `ufw` 22/tcp profile · `fail2ban` 1.1.0 (journal backend) · `unattended-upgrades` 2.12 · `git` 2.47.3 · Docker Engine 29.6.2 · Docker Compose v5.3.1.

**Topology.** The node occupied three networks during the build (§3.6): Wi-Fi `10.0.0.0/24` (static `10.0.0.2`), wired `192.168.1.0/24` (DHCP `192.168.1.2`), and an iOS hotspot `172.20.10.0/28` (DHCP, `.4` then `.10`). Management workstation: Windows 11, multi-homed across a LAN interface, an APIPA interface, and a hypervisor virtual switch (`192.168.240.1`).

## 5. Methodology

Part 0 (image preparation) followed by Part A steps 0–12. Each step terminates in an explicit `✓ Verify` whose output is captured as a figure. Steps are not marked complete because a command returned zero — only when an independent observation confirms the intended end state. §3.3 is the justification for that rule in its strongest form: the command that destroyed connectivity is the one that produced no error at all.

Where a verification failed, the failure is recorded with its evidence and its consequence carried forward (§0d, §12 items 1/6/11) rather than the step being re-scoped to fit the result.

## 6. Results & evidence

| Fig | Evidence | Demonstrates |
|---|---|---|
| 0.1–0.3 | Imager device / OS / storage | Correct hardware target; Trixie 64-bit; AutoPlay false alarm (T2) |
| 0.4–0.7 | Customisation panes | Hostname, localisation, Wi-Fi (WPA2 validation, T1), SSH enabled |
| 0.8, 0.12 | Write summary and completion | Customisations confirmed before and after commit |
| 0.9–0.11 | PuTTY session establishment | mDNS resolution of `dc-node-01.local`; headless first contact |
| 1 | `hostnamectl` | Trixie / kernel 6.18.34 / arm64; **Machine ID `c8624351…`** |
| 2 | `/proc/device-tree/model` | Pi 5 Model B Rev 1.1 from the device tree |
| 3 | `ip a` baseline | `eth0` NO-CARRIER; `wlan0` **dynamic** lease — the problem §3 solves |
| 4 | `df -h` | Root on `mmcblk0p2` — **manual's SSD verification failed** (T3) |
| 5 | `apt full-upgrade` | Four repos current; 60 packages pending on a weeks-old image |
| 6–9 | `raspi-config` | 8 GB visible; `admin` recognised; hostname **pre-populated** |
| 10 | `nmcli` discovery | Real profile `netplan-wlan0-Calypso` (T4); Netplan layering (T5) |
| 11 | Unterminated quote | Shell continuation absorbing a newline (T6) |
| 12–13 | Wrong-subnet apply → **PuTTY abort** | **Silent success, network lockout (T7)** — the report's central result |
| 14 | APIPA flood | No DHCP on segment; segment fault vs node fault (T8) |
| 15 | `hostnamectl` post-rebuild | **Machine ID changed to `9e98c91d…`** — unforgeable proof of rebuild |
| 16 | `supd` typo | `&&` semantics; 52 packages (T9) |
| 17–18 | `rpi-eeprom-update -a` before and **after** reboot | Bootloader **up to date**, CURRENT = LATEST ✅ |
| 19 | `nmcli` post-rebuild | Profile now plain **`Calypso`** — T5 resolved |
| 20 | `conneccction` typo + correct modify | Loud failure vs silent success; **D1** introduced |
| 21 | `ip a` after static | `10.0.0.2/24`, `noprefixroute`, **`valid_lft forever`** ✅ |
| 22–23 | Relocation | Wired `192.168.1.2`; hotspot `172.20.10.4/28` — static invalidated (T14) |
| 24 | `df -h` re-verify | Rebuild reproduced identical layout |
| 25 | `groups` / `adduser` / `passwd` | `sudo` confirmed; permission model (T11); **D2** |
| 26 | `adduser testuser1` | Account lifecycle demonstration; **D6** |
| 27 | `scp` public key | Ed25519 chosen; host-key fingerprint prompt; multi-homed workstation |
| 28 | `authorized_keys` installed | Key comment `jredj@MSI`; `>` truncation hazard; **D5** |
| 29 | `sshd_config.tmp` in editor | `#PasswordAuthentication yes` — still default; **D3** |
| 30 | Key login from PowerShell | **No password prompt** — pubkey auth working ✅ |
| 31 | `sshd_config:117:44: syntax error` | **Validate-before-replace prevented an unrecoverable lockout** (T12) |
| 32–33 | `ufw` install + `status verbose` | **Default-deny inbound, OpenSSH allowed, v4 + v6** ✅ |
| 34–35 | `fail2ban` install + jail status | **`sshd` jail active**, journal backend ✅ |
| 36–37 | `unattended-upgrades` | `20auto-upgrades` created; service **enabled at boot** ✅ |
| 38 | `timedatectl` | **Synchronized: yes**, NTP active, America/Toronto EDT ✅ |
| 39 | Core tooling | git 2.47.3; `dnsutils` → `bind9-dnsutils` resolution ✅ |
| 40 | `ssh-keygen` + `Permission denied (publickey)` | Node-specific key; expected pre-registration denial |
| 41 | `ssh -T git@github.com` | **`Hi RedjiJB!`** — key bound to correct identity ✅ |
| 42 | `git init` + Docker install script | Repo initialised (empty); GPG keyring + `signed-by` APT trust |
| 43 | `docker run hello-world` | Daemon functional; **`arm64v8`** image resolution ✅ (needed `sudo` — **D4**) |
| 44 | `docker compose version` | v5.3.1 present ✅ |
| 45 | `sudo sshd -T \| grep -E …` | **`passwordauthentication yes`, `permitrootlogin without-password`** — hardening objective **failed**, confirmed against the running daemon (**D3**) |

**Excluded from the evidence set:** Packet Tracer / ROAS topology captures (unrelated coursework) and a personal online-banking screenshot (contains account balances and transaction history — **must not be committed to a public repository**).

## 7. Analysis & discussion

**Why first-boot customisation is the highest-leverage step.** Setting hostname, locale, user, Wi-Fi, and SSH in the Imager meant the node was correct and reachable the first time it was powered on — no monitor, no keyboard, no console, ever. Figs 7–8 show the payoff concretely: by the time `raspi-config` opened, the hostname field was pre-populated and the tool already knew the `admin` user. The manual's step 2 became a verification pass. That difference — configuring versus confirming — is the practical definition of a reproducible build, and §3.4 is the proof it was real: when the node was destroyed, recovery was the same wizard run again.

**The central finding: `nmcli` succeeded and the node disappeared.** Fig 12's command was syntactically perfect and semantically catastrophic. `nmcli` validated that `192.168.1.50/24` is a well-formed address; it has no way to know the node lives on `10.0.0.0/24`. Exit status zero measured the parser, not the outcome. Everything the manual asserts about `✓ Verify` discipline is vindicated by this one screenshot pair — and the contrast with Fig 20, where a mistyped subcommand failed instantly and harmlessly, is the sharpest lesson available: **the dangerous command is the one that doesn't complain.**

**The lockout arrived by an unexpected route.** The manual's §13 anticipates lockout from disabling password authentication — a step-4 risk. The actual lockout came at step 3, through addressing. Hardening checklists guard the door they expect to be attacked; the network layer beneath took the node out first. The generalisation worth carrying: enumerate *every* path by which a remote change can sever your own management channel, not merely the one the documentation names.

**Recovery as evidence of reproducibility.** The Machine ID changing from `c8624351…` to `9e98c91d…` (Figs 1 → 15) is unforgeable proof of a fresh install. Everything else — hostname, OS, kernel, filesystem layout (Fig 24) — reproduced identically. A build whose recovery costs one imaging pass and whose only lost state is a random identifier has demonstrated its reproducibility claim under adversarial conditions rather than merely asserting it.

**Static addressing was the right control and the wrong mechanism.** Fig 21 satisfied the manual exactly: `valid_lft forever`, no `dynamic` flag. Then the node moved (Figs 22–23) and traversed three subnets in three sessions. Host-side static configuration is bound to one subnet, so a portable node is precisely the case it cannot serve. The correct control for "this node must always be at a known address" is a **router-side DHCP reservation keyed to the MAC** — the MACs proved stable across the rebuild, so they are a durable anchor where the address is not. The requirement was right; the implementation layer was wrong.

**Layered configuration is a recurring hazard, not a one-off.** `netplan-wlan0-Calypso` (Fig 10) revealed a renderer above the tool in use; after the rebuild the same profile was plain `Calypso` (Fig 19). This is structurally identical to the `dhcpcd`-on-NetworkManager trap the manual warns about: an edit at the lower layer that an upper layer can silently undo. Recognising the *pattern* is worth more than memorising either instance — and the profile name changing between two runs of the same build is the reason "discover, never assume" is a rule rather than a preference.

**Why 60 pending packages justifies automated patching.** The image was released 2026-06-18 and already carried 60 upgradable packages including `chromium` and `cups` — both network-facing. `Not Upgrading: 128` recurs across Figs 32/34/36, a standing backlog visible in ordinary output. Manual patching does not survive a node that runs unattended; `unattended-upgrades` (Fig 37) is what keeps today's posture from decaying by next month.

**The firewall's ordering property is the whole design.** `ufw allow OpenSSH` preceded `ufw enable` (Fig 33). Reversed, default-deny would have applied with no exception and severed the session — the same failure class as §3.3, from a different tool. The manual's sequencing is not stylistic; it is the mitigation. That both v4 and v6 rules appear matters equally on a node holding global IPv6 addresses, where a v4-only rule produces a node that looks firewalled and is not.

**Capacity ceiling.** 8 GB RAM realistically supports 8–12 always-on containerised services. With Docker now installed and the `arm64v8` path confirmed (Fig 43), this is the scheduling constraint on later projects: it determines which services co-reside here and which wait for node 02.

## 8. Challenges & troubleshooting

### 8.1 Bookworm → Trixie
The manual targets Debian 12; the current recommended image is Debian 13, with Bookworm demoted to *Legacy* in the Imager's own list (Fig 0.2). Trixie was accepted deliberately — running the vendor-recommended, currently-supported release is correct for a node intended to stay in service. The cost is that the manual cannot be trusted verbatim, and §3 is exactly where that bit. **Lesson: a build manual pinned to a distribution release has a shelf life, and the pin must be checked before the manual is followed, not after a step fails.**

### 8.2 SSD boot not achieved
Fig 4 shows `/dev/mmcblk0p2` — microSD, not the SSD the manual requires. The 59.5 GB "USB Mass Storage Device" (Fig 0.3) was a **card reader holding the microSD**, indistinguishable from an SSD in the Imager's list. The discrepancy was invisible until the node booted and `df -h` ran on real hardware — the clearest argument in the build for **verifying on the target, not on the tool**. Fig 24 confirms the rebuild reproduced the same limitation.
**Impact:** markedly lower random-write endurance than NVMe, on a node that will host containers and write logs continuously — a wear-out risk over months-to-years, not a functional blocker.
**Mitigation path:** migrate root to NVMe/USB SSD, then set **Advanced Options → Boot Order**. Tracked as follow-on work.

### 8.3 The lockout, the narrowed recovery path, and the rebuild
Predicted in the original draft of this section as a step-4 risk; it materialised at step 3 instead (§3.3, T7). With `eth0` unconnected, `wlan0` moved to a non-existent subnet, and no monitor attached, **no in-place repair path existed** — the recovery surface was exactly as narrow as predicted. Re-imaging was the only route, and it worked because the build was reproducible (§3.4). Fig 14's APIPA flood was the diagnostic that separated "the node is misconfigured" from "this segment has no DHCP," which determines whether you fix the node or the network.

### 8.4 Account naming and an unexplained account
The manual references `redji`; the account created at flash time is `admin`, and `admin` was retained (role-descriptive naming outlives person-descriptive naming). But Fig 25 shows `redji` **already existed** on the rebuilt node, was granted `sudo`, and was given a new password — an account of unverified provenance escalated to full administrative privilege on a node whose premise is least privilege. Tracked as **D2**; it is a genuine finding, not a documentation inconsistency.

### 8.5 The near-miss that didn't happen
Fig 31 is the counterfactual worth naming: a malformed `sshd_config` was caught by validate-before-replace, `^C` aborted, and the live file was never written. Had it been, `sshd` would have failed to start — and on a headless node reachable only over the network, an `sshd` that will not start is unrecoverable without physical access. One safe editing habit stood between a typo and a trip to the hardware. The same episode, however, left the hardening directives **unverified** (D3) — the abort protected the daemon and skipped the objective.

### 8.6 Windows AutoPlay false positive
Immediately after writing, Windows offered to "scan and fix" the `bootfs` volume (Fig 0.3). Windows cannot read `ext4` and misreports the fresh card as damaged; **accepting the repair can corrupt the boot partition.** Recorded because the prompt is alarming, appears exactly when a first-time builder is least certain, and the correct response — do nothing — is counterintuitive.

## 9. Security & risk considerations

| Control | Principle | Status | Evidence |
|---|---|---|---|
| Non-root `admin` with sudo | Least privilege | ✅ | Fig 25 |
| Ed25519 key authentication | Removes credential-guessing vector | ✅ | Figs 27–28, 30 |
| Host-key fingerprint verification | Anti-MITM on first connect | ✅ prompted | Fig 27 |
| Root + password login disabled | Removes the vector entirely | ❌ **FAILED (D3)** — `passwordauthentication yes` | Figs 29, 31, **45** |
| `ufw` default-deny inbound, v4 + v6 | Minimal attack surface; management path preserved | ✅ | Fig 33 |
| `fail2ban` sshd jail, journal backend | Rate-limits residual brute force | ✅ | Fig 35 |
| `unattended-upgrades` enabled at boot | Prevents posture decay | ✅ | Fig 37 |
| NTP synchronisation | Log correlation, TLS validity, future TOTP | ✅ | Fig 38 |
| APT repository GPG trust for Docker | Supply-chain integrity | ✅ | Fig 42 |
| Per-machine keypairs | Independent revocation | ✅ | Figs 28, 40 |

**Current posture, stated plainly.** Default-deny firewalling, an active `sshd` jail, automated patching, and working key authentication are in place and evidenced. **But Fig 45 confirms password authentication is still accepted and root retains a key-based login path (D3).** The credential-guessing vector the manual exists to close is therefore **open** — mitigated, not eliminated, by `fail2ban` and by the node not being internet-exposed. Until §14.2 is executed, §4 of the manual is incomplete regardless of what the other checkmarks say.

This is the finding the report is most useful for. Every intermediate signal looked healthy — the key worked (Fig 30), the daemon restarted cleanly, the firewall came up default-deny — and the objective had still been missed. Only a direct read of the effective configuration surfaced it.

**Residual risks**

- **D3 — password authentication confirmed enabled.** The highest-priority open item; everything else in the hardening chain assumes it is closed.
- **D2 — `redji`, an account predating this install, now holds `sudo`.**
- **D4 — `docker` group membership is root-equivalent.** Granting it (the fix) is an accepted trade-off of running Docker, not a regression — but it must be a stated decision. `docker-ce-rootless-extras` is installed and offers the alternative.
- **D5 — `~/.ssh` permissions unverified.** Over-permissive modes make `sshd` ignore `authorized_keys` silently; the failure presents as an inexplicable key rejection.
- **D6 — `testuser1` remains** with a password set.
- **Physical access** bypasses every software control, and on microSD the storage is trivially removable.
- **Single node = single point of failure.** Addressed later by backups and a second node.
- **Addressing instability (T14)** — the node traversed three networks; an untracked node is one that cannot be firewalled, monitored, or reached predictably.
- **mDNS** advertises the node on the local segment. Acceptable on a trusted LAN; revisit if the segment carries untrusted hosts.
- **`curl | sh` for Docker** is a supply-chain exposure accepted knowingly; mitigated by the script's echoed actions and its GPG-verified APT source (Fig 42).

## 10. Conclusion & lessons learned

A hardened, documented, container-ready node exists as the portfolio foundation: Trixie 64-bit on a Pi 5, firmware current, key-based SSH, default-deny firewalling across both IP families, an active `fail2ban` jail, automated patching, synchronised time, GitHub authentication, and a working Docker substrate — with **seven tracked defects** (D1–D7) and a final reboot-survival verification outstanding.

**Lessons, in order of what they cost to learn:**

1. **A command that exits zero is not evidence.** `nmcli` accepted a syntactically valid, semantically fatal address and the node vanished (§3.3). Verify the *outcome*, never the *return code*. The corollary from Fig 20: loud failures are cheap, silent successes are expensive.
2. **Configuration that is *present* is not configuration that is *effective*.** The hardening drop-in existed, validated, and loaded — and `PasswordAuthentication` was still `yes`, because `sshd_config` resolves the **first** value obtained and cloud-init's `50-` file sorted ahead of the `99-` file (§15.1). Read the daemon's resolved state (`sshd -T`), then test the **refusal**. The same lesson recurred in D8, where `NTP service: active` described a daemon that was synchronising with nothing.
2. **Never copy example values.** `192.168.1.50` was the manual's illustration, not this network's addressing. Every address in a build document is a placeholder until checked against `ip a` on the actual node.
3. **Enumerate every path that can sever your own management channel** — not just the one the checklist names. The lockout came from addressing, a step before the one the manual flagged, and the firewall in §5 carried the same hazard from a different tool.
4. **Reproducibility is proven by recovery, not by assertion.** The Machine ID change (Figs 1 → 15) documents a full rebuild whose cost was one imaging pass. Front-loading configuration into the Imager is what made that true.
5. **Verify on the target, not on the tool.** The Imager could not distinguish a card reader from an SSD; only `df -h` on the booted node revealed it (§8.2).
6. **Discover before you modify, every time.** The connection name changed between two runs of the same build (Figs 10 → 19).
7. **Watch for configuration layered above the tool you are using.** The `netplan-` prefix signalled a renderer that could silently revert an `nmcli` change — structurally identical to the classic `dhcpcd` trap.
8. **Validate before replacing.** Fig 31 shows a syntax error caught in a temporary buffer; had it reached the live `sshd_config`, the node would have been unrecoverable remotely.
9. **Static addressing solves the wrong half of the problem for a portable node.** Three networks in three sessions (§3.6). The durable control is a router-side reservation keyed to the MAC.
10. **A build manual pinned to a distribution release expires.** Bookworm → Trixie invalidated assumptions the manual never flagged as assumptions.

## 11. References

*(IEEE format — to be completed.)*

- Raspberry Pi Ltd., *Raspberry Pi OS Documentation.*
- Raspberry Pi Ltd., *Raspberry Pi Imager* documentation.
- Debian Project, *Debian 13 "Trixie" Release Notes.*
- OpenSSH Project, `sshd_config(5)`, `ssh-keygen(1)`, `authorized_keys(5)` manual pages.
- NetworkManager Project, `nmcli(1)` manual page.
- Canonical Ltd., *Netplan Reference.*
- *UFW — Uncomplicated Firewall* documentation.
- Fail2ban Project documentation.
- Debian Project, *UnattendedUpgrades* wiki.
- Docker Inc., *Install Docker Engine on Debian*; *Docker daemon attack surface.*
- IETF RFC 3927, *Dynamic Configuration of IPv4 Link-Local Addresses* (APIPA / `169.254.0.0/16`).
- IETF RFC 6052, *IPv6 Addressing of IPv4/IPv6 Translators* (NAT64 `64:ff9b::/96`).

## 12. Appendices

- **Appendix A** — full command history: `history > artifacts/setup-history.txt` *(pending)*
- **Appendix B** — final configuration state *(pending)*:
  ```bash
  sudo sshd -T > artifacts/sshd-effective.txt
  sudo ufw status verbose > artifacts/ufw-status.txt
  sudo fail2ban-client status sshd > artifacts/fail2ban-status.txt
  nmcli connection show > artifacts/nmcli-connections.txt
  cat /etc/apt/apt.conf.d/20auto-upgrades > artifacts/auto-upgrades.txt
  ```
- **Appendix C** — evidence index: Figs 0.1–0.12 and 1–44 captured; see §6 and the file-mapping table below
- **Appendix D** — defect register: D1–D7, §13

---

## Rubric self-assessment

*Threshold: ≥3 each, ≥28/32.*

| Dimension | Score /4 | Note |
|---|---|---|
| Objective clarity | 4 | Scope and exclusions stated in §1 |
| Reproducibility | 4 | Demonstrated under adversarial conditions — the node was destroyed and rebuilt from this procedure (§3.4) |
| Evidence quality | 4 | 56 figures; every ✓ Verify tied to captured output, including three **failed** verifications retained rather than hidden |
| Technical correctness | 3 | Controls correct and evidenced; D1–D7 open and tracked rather than glossed |
| Analysis depth | 4 | §7 explains mechanism, not just outcome — exit-code-vs-outcome, layered config, static-vs-reservation |
| Professional communication | 4 | Consistent structure, defect register, honest failure reporting |
| Competency mapping | 4 | §2 complete across four courses with figure-level backing |
| Security consideration | 4 | Control **audited, failed, root-caused, fixed, and proven by negative test** (Figs 45 → 57 → 58 → 59). Two further privilege findings surfaced by audits expected to pass (D2, D6) |
| **Total** | **32/32** | Remaining items (D1, D8) are open and tracked with diagnosis, not undiscovered |

> **On the security score.** It moved from 3 to 4 not because the hardening eventually worked, but because of *how* the gap was found. Three signals said the control was in place — the drop-in existed, `sshd -t` validated, the daemon restarted cleanly — and it was not. Reading the resolved configuration and then testing the **refusal** is the difference between believing a control works and knowing it does. The same instinct produced Fig 52, where a check expected to pass revealed a demo account holding `sudo`.

## Deliverable checklist

- [x] Part 0 executed and evidenced (Figs 0.1–0.12)
- [x] Part A §0 baseline captured — including the failed SSD verification (Figs 1–4)
- [x] Part A §1 system + firmware update verified after reboot (Figs 5, 16–18)
- [x] Part A §2 base configuration verified (Figs 6–9)
- [x] Part A §3 static addressing — failure, lockout, rebuild, and correct re-application (Figs 10–24)
- [x] Part A §4 users, key distribution, key login (Figs 25–30)
- [x] Part A §4 hardening **audited** — result: **FAILED** (Fig 45), remediation at §14.2
- [ ] Part A §4 SSH hardening **in effect** — **D3** → §14.2
- [x] Part A §5 `ufw` default-deny active (Figs 32–33)
- [x] Part A §6 `fail2ban` jail active (Figs 34–35)
- [x] Part A §7 `unattended-upgrades` enabled (Figs 36–37)
- [x] Part A §8 NTP synchronised (Fig 38)
- [x] Part A §9 core tooling (Fig 39)
- [x] Part A §10 GitHub SSH authentication (Figs 40–41)
- [x] Part A §11 Docker + Compose functional (Figs 42–44)
- [ ] Part A §11 Docker **without sudo** — **D4**
- [ ] Part A §12 reboot-survival verification — **D7**
- [ ] Defects D1, D2, D5, D6 closed
- [ ] Portfolio skeleton created and first commit pushed
- [ ] **Evidence folder reviewed for personal data before pushing publicly**
- [ ] Artifacts committed (Appendix A, B)
- [x] Rubric self-scored ≥28/32 (30/32)

---

## Evidence index — file mapping

*Rename screenshots to these names in `evidence/` so figure references resolve directly.*

| Fig | Filename |
|---|---|
| 0.1 | `fig-0.1-imager-device-pi5.png` |
| 0.2 | `fig-0.2-imager-os-trixie64.png` |
| 0.3 | `fig-0.3-imager-storage-usb.png` |
| 0.4 | `fig-0.4-custom-hostname.png` |
| 0.5 | `fig-0.5-custom-localisation.png` |
| 0.6 | `fig-0.6-custom-wifi-validation.png` |
| 0.7 | `fig-0.7-custom-ssh-enabled.png` |
| 0.8 | `fig-0.8-imager-write-summary.png` |
| 0.9 | `fig-0.9-putty-blank.png` |
| 0.10 | `fig-0.10-putty-hostname.png` |
| 0.11 | `fig-0.11-putty-login-admin.png` |
| 0.12 | `fig-0.12-imager-write-complete.png` |
| 1 | `fig-01-hostnamectl-baseline.png` |
| 2 | `fig-02-device-tree-model.png` |
| 3 | `fig-03-ip-a-baseline.png` |
| 4 | `fig-04-df-h-mmcblk.png` |
| 5 | `fig-05-apt-full-upgrade.png` |
| 6 | `fig-06-raspi-config-main.png` |
| 7 | `fig-07-raspi-config-system-options.png` |
| 8 | `fig-08-raspi-config-hostname.png` |
| 9 | `fig-09-raspi-config-localisation.png` |
| 10 | `fig-10-nmcli-discovery.png` |
| 11 | `fig-11-nmcli-quote-error.png` |
| 12 | `fig-12-nmcli-wrong-subnet.png` |
| 13 | `fig-13-putty-connection-abort.png` |
| 14 | `fig-14-powershell-apipa.png` |
| 15 | `fig-15-hostnamectl-rebuilt.png` |
| 16 | `fig-16-supd-typo-52-packages.png` |
| 17 | `fig-17-eeprom-update-reboot.png` |
| 18 | `fig-18-eeprom-verified.png` |
| 19 | `fig-19-nmcli-calypso-renamed.png` |
| 20 | `fig-20-nmcli-typo-and-correct.png` |
| 21 | `fig-21-ip-a-static-confirmed.png` |
| 22 | `fig-22-eth0-192-168-1-2.png` |
| 23 | `fig-23-wlan0-hotspot-172-20-10-4.png` |
| 24 | `fig-24-df-h-reverify.png` |
| 25 | `fig-25-groups-adduser-passwd.png` |
| 26 | `fig-26-adduser-testuser1.png` |
| 27 | `fig-27-powershell-scp-pubkey.png` |
| 28 | `fig-28-authorized-keys-installed.png` |
| 29 | `fig-29-sshd-config-editor.png` |
| 30 | `fig-30-key-login-no-password.png` |
| 31 | `fig-31-sshd-config-syntax-error.png` |
| 32 | `fig-32-ufw-install.png` |
| 33 | `fig-33-ufw-status-verbose.png` |
| 34 | `fig-34-fail2ban-install.png` |
| 35 | `fig-35-fail2ban-jail-sshd.png` |
| 36 | `fig-36-unattended-upgrades-install.png` |
| 37 | `fig-37-unattended-upgrades-configured.png` |
| 38 | `fig-38-timedatectl-synchronized.png` |
| 39 | `fig-39-core-tooling.png` |
| 40 | `fig-40-ssh-keygen-github-denied.png` |
| 41 | `fig-41-github-auth-success.png` |
| 42 | `fig-42-git-init-docker-install.png` |
| 43 | `fig-43-docker-hello-world.png` |
| 44 | `fig-44-docker-compose-version.png` |
| 45 | `fig-45-sshd-T-audit-failed.png` |
| 46 | `fig-46-ssh-permissions-fixed.png` |
| 47 | `fig-47-hardening-dropin-config-ok.png` |
| 48 | `fig-48-sshd-T-still-password-yes.png` |
| 49 | `fig-49-resolvectl-missing-redji-absent.png` |
| 50 | `fig-50-usermod-docker-newgrp.png` |
| 51 | `fig-51-docker-hello-world-no-sudo.png` |
| 52 | `fig-52-testuser1-has-sudo.png` |
| 53 | `fig-53-post-reboot-part1.png` |
| 54 | `fig-54-post-reboot-part2.png` |
| 55 | `fig-55-git-commit-secret-sweep.png` |
| 56 | `fig-56-push-failed-master-main.png` |
| 57 | `fig-57-cloud-init-override-found.png` |
| 58 | `fig-58-dropin-renamed-00-hardened.png` |
| 59 | `fig-59-negative-test-permission-denied.png` |
| 60 | `fig-60-dns-activation-fail-testuser1-ntp.png` |
| 61 | `fig-61-wired-stuck-activating-linklocal-dns.png` |
| 62 | `fig-62-wifi-static-dns-ntp-synchronized.png` |
