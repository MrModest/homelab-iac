# Debian 13 vs Ubuntu Server 26.04 LTS

Research for issue #18. This covers facts and trade-offs only. It makes no recommendation. Gathered 2026-10-23.
Items marked **[unverified]** were not confirmed against a primary source. Package versions come from the `amd64` Packages indexes of `deb.debian.org` (trixie, trixie-updates, trixie-security), `archive.ubuntu.com` (resolute, resolute-updates) and `download.docker.com`, all read on 2026-10-23.

ZFS details come from the earlier findings in [zfs-and-os.md](https://github.com/MrModest/homelab-iac/blob/research/zfs-and-os/docs/research/zfs-and-os.md) (issue #5). This doc summarises them and does not repeat them.

## Summary table

| | Debian 13 "trixie" | Ubuntu Server 26.04 LTS "resolute" |
|---|---|---|
| Released | 13.0 on 2025-08-09. Latest point release 13.7 on 2026-09-12 | 2026-04-23. 26.04.1 on 2026-08-27 |
| Standard security support | to 2028-08-09 (Debian Security team) | to May 2031 (`main` only) |
| Extended support | LTS 2028-08-09 to 2030-06-30 (free, volunteer and sponsor funded, reduced arch set). ELTS by Freexian 2030-07-01 to 2035-06-30 (commercial) | ESM via Ubuntu Pro to May 2036 (`main` + `universe`). Legacy add-on to May 2041 |
| Cadence | about every 2 years, no fixed date | LTS every 2 years (April). Interims every 6 months |
| Kernel | 6.12 LTS series (`linux-image-amd64` 6.12.113-1 in security). Backports kernel up to 7.2 | GA 7.0 (`linux-image-generic` 7.0.0-38.38). HWE optional on server |
| OpenZFS | 2.3.9, `zfs-dkms` (contrib), built locally | 2.4.1, prebuilt and signed, in `linux-main-modules-zfs-*` |
| Secure Boot + ZFS | needs a DKMS MOK key enrolled by hand at the console | works out of the box (Canonical-signed module) |
| systemd | 257.13 | 259.9 |
| Samba | 4.22.11 | 4.23.6 |
| OpenSSH / Python / APT | 10.0p1 / 3.13.5 / 3.0.3 | 10.2p1 / 3.14.3 / 3.2.0 |
| Docker CE repo | supported (trixie), docker-ce 29.9.2 | supported (resolute), docker-ce 29.9.2 |
| Distro Docker package | `docker.io` 26.1.5 (main) | `docker.io` 29.1.3 (universe) |
| sudo / coreutils | C sudo 1.9.16p2, GNU coreutils 9.7 | **sudo-rs** 0.2.13 default; **rust-coreutils** 0.10 default (GNU `cp df mv rm true`) |
| Network config on a plain install | ifupdown (`/etc/network/interfaces`) **[unverified, inferred from installer logic]** | netplan 1.2 → systemd-networkd |
| Preinstalled extras | none (minimal "standard system utilities" task) | snapd, cloud-init, systemd-resolved, chrony; `ubuntu-server` also Recommends landscape-common, lxd-agent-loader, needrestart, unattended-upgrades |
| unattended-upgrades | the installer defaults to **not** installing it (`pkgsel/update-policy` default `none`) | installed (Recommended) and enabled. Origins: release, `-security`, ESM-apps, ESM-infra |
| Upsell | none | Pro/ESM ads in `apt upgrade` output and MOTD (removable) |
| Ansible `password` become | works | **broken on ansible-core ≤ 2.21** (sudo-rs prompt). Fixed only on `devel` / 2.22 |

## 1. Support lifetime, cadence, upgrade path

### Debian 13

- 13.0 released **2025-08-09**. The latest point release is **13.7, 2026-09-12** ([debian.org/releases/trixie](https://www.debian.org/releases/trixie/)).
- The Debian Security team gives full security support for 3 years, to **2028-08-09**. Debian LTS follows from **2028-08-09 to 2030-06-30**. LTS architectures for trixie are still "To be determined" ([wiki.debian.org/LTS](https://wiki.debian.org/LTS)).
- ELTS is a commercial service from Freexian ("not an official Debian project"). It is scheduled for trixie from **2030-07-01 to 2035-06-30**. It is free on request for individuals and small non-profits ([wiki.debian.org/LTS/Extended](https://wiki.debian.org/LTS/Extended)).
- Cadence: a new stable release comes "every couple of years", with a freeze of about 7 months. Users get about 1 year to upgrade after the next stable release ships ([wiki.debian.org/DebianReleases](https://wiki.debian.org/DebianReleases)).
- Upgrade path: only 12 → 13 is supported, with `apt full-upgrade` (optionally `apt upgrade --without-new-pkgs` first). `apt modernize-sources` converts sources to deb822. Run `apt purge '?obsolete'` afterwards ([release notes, upgrading](https://www.debian.org/releases/trixie/release-notes/upgrading.en.html)). Point releases are not separate upgrades. They arrive through the normal archive.
- An Ubuntu → Debian in-place conversion is not a supported path in either distro's docs **[unverified, no doc found that covers it]**. A switch means a reinstall.

### Ubuntu 26.04 LTS

- Released **2026-04-23**. 26.04.1 was announced **2026-08-27** ([ubuntu-announce](https://lists.ubuntu.com/archives/ubuntu-announce/2026-August/000326.html)).
- Standard security maintenance runs to **May 2031** and covers `main`. ESM (Ubuntu Pro) runs to **May 2036** and adds `universe`. The Legacy add-on runs to **May 2041** ([ubuntu.com/about/release-cycle](https://ubuntu.com/about/release-cycle)). Ubuntu Pro is free for personal use on a small number of machines **[unverified, number not re-checked here]**.
- Cadence: an LTS every 2 years with 5 years of standard support. Interims every 6 months with 9 months of support (same source).
- Upgrade path: "you must first upgrade to either Ubuntu 24.04 LTS or 25.10 before you can proceed to 26.04 LTS" ([26.04 release notes](https://documentation.ubuntu.com/release-notes/26.04/)). 22.04 hosts therefore need two hops (22.04 → 24.04 → 26.04).
- 24.04 → 26.04 via `do-release-upgrade` was held back until after 26.04.1 "for planned backports to address regressions in a recent version of rust-coreutils" (26.04.1 announcement). `resolute` / 26.04.1 is now listed with `Supported: 1` in [changelogs.ubuntu.com/meta-release-lts](https://changelogs.ubuntu.com/meta-release-lts), which is the file `do-release-upgrade` reads. The enablement date of 2026-09-29 comes from secondary reporting only ([ubuntuhandbook](https://ubuntuhandbook.org/index.php/2026/09/ubuntu-24-04-to-ubuntu-26-04-1-upgrade/)) **[unverified]**.
- Upgrade notes from 24.04 that matter for a server ([summary for LTS users](https://documentation.ubuntu.com/release-notes/26.04/summary-for-lts-users/)):
  - cgroup v1 removed.
  - `apt-key` removed.
  - Chrony replaces systemd-timesyncd on new installs (upgraders do this manually).
  - Dracut is the default initramfs generator (initramfs-tools is still supported).
  - SSSD runs as the `sssd` user.
  - Samba disables NetBIOS by default on fresh installs and enables SMB3 Unix Extensions by default.
  - 26.04 is the last release with SysV init-script compatibility.

## 2. Kernel, ZFS, Secure Boot

### Debian

- Ships the **6.12** series, the upstream LTS kernel ([what's new](https://www.debian.org/releases/trixie/release-notes/whats-new.en.html)). It is updated in place through security and point releases (6.12.107-1 in trixie, 6.12.113-1 in trixie-security). There is no HWE concept. Newer kernels come from `trixie-backports` (up to 7.2 per #5).
- ZFS: `zfs-dkms` + `zfsutils-linux` 2.3.9 from **contrib**, compiled on every kernel install. A failed build means pools don't import on the new kernel. A backports kernel can outrun the stable ZFS (see #5).
- Secure Boot: supported out of the box via Microsoft-signed shim, and kernels are signed. DKMS modules are signed with a MOK at `/var/lib/dkms/mok.{key,pub}`, generated automatically on the first build. Enrollment (`mokutil --import`) must be confirmed "directly from the console at boot time" ([wiki.debian.org/SecureBoot](https://wiki.debian.org/SecureBoot)). This is one manual console step per host. It cannot be done by Ansible alone.
- Debian 13 admin-visible changes ([issues to be aware of](https://www.debian.org/releases/trixie/release-notes/issues.en.html)):
  - `/tmp` is tmpfs (up to 50% of RAM). Revert with `systemctl mask tmp.mount`.
  - systemd-tmpfiles cleans `/tmp` after 10 days and `/var/tmp` after 30. On upgrades, a `tmp.conf` override keeps the old behaviour.
  - `systemd-sysctl` no longer reads `/etc/sysctl.conf` (use `/etc/sysctl.d/`).
  - Interface names may change on upgrade.
  - `/boot` needs at least 768 MB.
  - `last`/`lastlog` are removed.

### Ubuntu

- GA kernel **7.0** for 26.04 (7.0.0-38.38 in -updates). Server installs "default to the GA kernel and provide the enablement kernel as optional" ([kernel lifecycle](https://ubuntu.com/kernel/lifecycle)). HWE kernels roll forward with each interim (for example, 24.04.4 HWE = 6.17, 24.04.5 HWE = 7.0). The 26.10 interim kernel is 7.3. The GA 7.0 kernel is supported to May 2031 (standard) and May 2036 (ESM).
- ZFS: OpenZFS **2.4.1** userspace in `main`. The kernel module is prebuilt and signed in `linux-main-modules-zfs-<kver>`, pulled in by `linux-image-generic`. There is no DKMS step and no MOK step, so Secure Boot works as-is (from #5). Known caveat from #5: HWE kernels can bring a newer module than the userspace (seen on 24.04: 2.2.2 tools with a 2.4.1 HWE module).

## 3. Docker, Samba, systemd

- Docker's official repo supports both: Debian **Trixie 13 (stable)** and Bookworm 12 ([docs](https://docs.docker.com/engine/install/debian/)), and Ubuntu **Resolute 26.04 (LTS)**, Noble 24.04, Jammy 22.04 and Stonking 26.10 ([docs](https://docs.docker.com/engine/install/ubuntu/)). Both repos carry **docker-ce 5:29.9.2**, docker-compose-plugin **5.6.1** and containerd.io **2.4.2** as of 2026-10-23.
- Docker caveats are the same on both. Published ports bypass ufw/firewalld. Only `iptables-nft`/`iptables-legacy` are supported ("rules created with `nft` are not supported"), so rules go in the `DOCKER-USER` chain. Distro packages (`docker.io`, `podman-docker`, `containerd`, `runc`, …) must be removed first.
- Distro-packaged Docker: Debian `docker.io` 26.1.5+dfsg1-9+deb13u1 (main, security-supported). Ubuntu `docker.io` 29.1.3-0ubuntu4.1 (universe, so standard support is not guaranteed and security updates come only with ESM/Pro).
- Samba: Debian **4.22** (2:4.22.11+dfsg-0+deb13u1). Ubuntu **4.23** (2:4.23.6+dfsg-1ubuntu2.2). Debian tracks upstream Samba minor releases in stable (4.22.10 → 4.22.11 in security/point releases).
- systemd: Debian **257** (257.13-1~deb13u1). Ubuntu **259** (259.9-0ubuntu0.1). Ubuntu 26.04 removed cgroup v1 support from systemd. The Debian status of cgroup v1 is **[unverified]**, though Docker uses cgroup v2 on both by default.

## 4. Default footprint and opinionated extras

### Ubuntu Server 26.04

The metapackages (version 1.570) define the default footprint:
- `ubuntu-server-minimal` **Depends** on `cloud-init`, `snapd`, `sudo-rs`, `systemd-resolved`, `chrony | time-daemon`, `ubuntu-release-upgrader-core`, `unminimize`, `lvm2`, `mdadm`, `multipath-tools`, `open-iscsi`, `btrfs-progs` and others.
- `ubuntu-server-minimal` **Recommends** `unattended-upgrades`, `needrestart`, `kdump-tools`, `hwctl`.
- `ubuntu-server` additionally Depends on `lxd-installer`, `motd-news-config`, `pollinate`, `overlayroot`, `cloud-initramfs-*`, and Recommends `snapd`, `landscape-common`, `lxd-agent-loader`, `fwupd`, `open-vm-tools`.

Removing snapd or cloud-init removes the metapackage. Whether that has side effects on future upgrades is **[unverified]**. Whether the ISO installer disables cloud-init after first boot is **[unverified]**.

Other Ubuntu-specific behaviour:
- **Networking:** netplan 1.2 rendering to systemd-networkd.
- **sudo-rs** is the default `sudo`. The original is still installable as `sudo.ws`. `sudoers.ldap` and I/O logging are not supported ([server docs](https://ubuntu.com/server/docs/reference/other-tools/sudo-rs/)).
- **rust-coreutils** is the default (0.10.0 in -updates). `cp`, `df`, `mv`, `rm` and `true` still come from GNU in 26.04, and none will from 26.10 on. Reverting needs `apt install coreutils-from-gnu coreutils-from-uutils- --allow-remove-essential` (same doc).
- **unattended-upgrades** 2.12ubuntu9: the debconf `enable_auto_updates` default is `true`. Allowed-Origins are `${distro_codename}`, `-security`, `ESMApps:-apps-security` and `ESM:-infra-security`. `Automatic-Reboot` is off (commented out). Read from the .deb.
- **Ubuntu Pro messaging:**
  - On unattached machines, `apt upgrade` prints "Get more security updates through Ubuntu Pro with 'esm-apps' enabled:" via `/etc/apt/apt.conf.d/20apt-esm-hook.conf` ([Pro client docs](https://ubuntu.com/pro-client/docs//en/latest/explanations/apt_messages/)).
  - APT News can be disabled with `pro config set apt_news=false`. ESM lines in the MOTD can be hidden with `/var/lib/update-notifier/hide-esm-in-motd` ([Pro client how-to](https://documentation.ubuntu.com/pro-client/en/latest/howtoguides/disable_enable_apt_news/)).
  - Whether `apt_news=false` also suppresses the esm-apps hook message is **[unverified]**. A 2025 bug about an unsuppressible message was closed Invalid ([LP#2110353](https://bugs.launchpad.net/bugs/2110353)).

### Debian 13

- No snapd, cloud-init or netplan on an installer-based install (they are available in the archive: snapd 2.68.3, cloud-init 25.1.4, netplan.io 1.1.2).
- **Networking:** the installer (`netcfg`) writes netplan config only if `netplan-generator` is in the target, NetworkManager config if NM is present, and otherwise falls back to **ifupdown** `/etc/network/interfaces` ([netcfg changelog 1.190](https://salsa.debian.org/installer-team/netcfg/-/raw/master/debian/changelog)). For a no-desktop install this implies ifupdown **[unverified, not tested]**. The Debian wiki says systemd-networkd "is the default network manager for non-graphical systems" ([wiki](https://wiki.debian.org/NetworkConfiguration)), which conflicts with the installer logic. Treat that as wiki guidance, not installer behaviour. ifupdown users should use `dhcpcd-base` in place of `isc-dhcp-client` (13 release notes).
- **unattended-upgrades:** the installer question `pkgsel/update-policy` defaults to **`none`** ("By default, security updates are not automatically installed") ([pkgsel templates, master](https://salsa.debian.org/installer-team/pkgsel/-/raw/master/debian/pkgsel.templates)). The package itself (2.12) has debconf default `true` once installed. Its Origins-Pattern covers `label=Debian` (point-release updates) plus `Debian-Security`. `Automatic-Reboot` is off.
- **sudo:** C sudo 1.9.16p2. **coreutils:** GNU 9.7.

## 5. Security update practice

- Debian:
  - Security support is run by the Debian Security team via DSAs (and DLAs once in LTS), tracked at [security-tracker.debian.org](https://security-tracker.debian.org).
  - Documented exceptions in 13: qtwebengine-based apps are not covered, and statically linked Go/Rust packages get "limited security support", mostly updated in point releases ([release notes](https://www.debian.org/releases/trixie/release-notes/issues.en.html)).
  - The whole `main` archive gets security support with no paywall. `contrib` (where ZFS lives) is not part of `main`. Its security support status is **[unverified]**, but the zfs-linux 2.3.9 update did ship via trixie-security (#5).
- Ubuntu:
  - USNs, tracked at ubuntu.com/security.
  - Free standard support covers **`main` only**. **`universe`** security fixes (which include `docker.io`, `sanoid` and many homelab tools) need ESM-Apps, which needs Ubuntu Pro ([release-cycle](https://ubuntu.com/about/release-cycle)).
- One comparable data point is CVE-2024-6387 (regreSSHion, OpenSSH). Both published fixes the same day, 2024-07-01: [DSA-5724-1](https://lists.debian.org/debian-security-announce/2024/msg00135.html) and [USN-6859-1](https://ubuntu.com/security/notices/USN-6859-1). No primary-source dataset comparing time-to-patch across distros was found. Any broader claim is **[unverified]**.
- Docker CE from download.docker.com is patched by Docker on the same schedule for both distros. It falls outside both distros' security teams.

## 6. Community signals and recent controversies

- Debian is the base of Proxmox VE 9 (9.0 on Debian 13, Aug 2025; 9.2 on Debian 13.5, May 2026, with kernel 7.0 and ZFS 2.4) ([Proxmox press release](https://www.proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-0)). Proxmox ships its own kernel and ZFS rather than Debian's. No primary-source usage statistics for homelab Ubuntu vs Debian servers were found **[unverified]**.
- Ubuntu, rust-coreutils:
  - In 25.10 the Rust `date -r` returned the wrong date. This broke the unattended-upgrades periodic check ([LP#2127970](https://bugs.launchpad.net/bugs/2127970), Fix Released for questing and resolute).
  - Rust-coreutils regressions are also the stated reason for delaying 24.04 → 26.04 upgrades until after 26.04.1.
- Ubuntu, sudo-rs: its prompt handling broke Ansible's `sudo` become plugin, see §7.
- Ubuntu, Pro/ESM upsell: advertising in `apt` output has drawn recurring complaints since 2022 ([Discourse thread](https://discourse.ubuntu.com/t/remove-pro-advertising-from-apt/34051)).
- Ubuntu, snap: snapd is a hard dependency of `ubuntu-server-minimal`, and LXD installs as a snap (`lxd-installer`).
- Debian:
  - Trixie's `/tmp`-on-tmpfs and automatic `/tmp` cleanup are behaviour changes for upgraders (§2).
  - ZFS remains in contrib with DKMS because of licensing.
  - No Debian controversy of comparable operational impact was found for trixie **[unverified, absence of evidence]**.

## 7. Ansible-relevant differences

- **Facts:**
  - Both report `ansible_os_family: Debian`.
  - `ansible_distribution` is `Debian` vs `Ubuntu`, and `ansible_distribution_release` is `trixie` vs `resolute`.
  - `ansible_distribution_major_version` is `13` vs `26`. Debian's `ansible_distribution_version` includes the point release (e.g. `13.7`) **[unverified, not run]**.
  - Ubuntu 26.04 support was added to ansible-core CI in [ansible/ansible#86894](https://github.com/ansible/ansible/pull/86894).
- **Python on the target:** 3.13 (Debian) vs 3.14 (Ubuntu). ansible-core 2.20 and 2.21 support target Python 3.9–3.14 ([support matrix](https://docs.ansible.com/projects/ansible/latest/reference_appendices/release_and_maintenance.html)).
- **sudo-rs vs `become: true` (Ubuntu only):**
  - sudo-rs wraps Ansible's `-p` prompt inside its own `[sudo: … ]` prefix, so password-based become times out ([ansible/ansible#85837](https://github.com/ansible/ansible/issues/85837), [sudo-rs#1461](https://github.com/trifectatechfoundation/sudo-rs/issues/1461), still open).
  - The fix ([#86175](https://github.com/ansible/ansible/pull/86175)) merged to `devel` on 2026-04-27, and a follow-up handles sudo-rs i18n (#86964, 2026-09-30).
  - The 2.20 backport was merged and then **reverted** the same day (2026-05-11). The 2.21 backport PR was closed unmerged (2026-10-07). Released `v2.20.10` and `v2.21.5` contain no sudo-rs handling. The fix ships from **2.22** (`v2.22.0rc1` tagged).
  - Workarounds on ≤ 2.21: NOPASSWD sudoers for the Ansible user, `become_exe: sudo.ws`, or `update-alternatives --set sudo /usr/bin/sudo.ws`.
  - Debian 13 uses C sudo and is unaffected.
- **Packages and paths that differ:**
  - Network config: netplan YAML (`/etc/netplan/`) vs `/etc/network/interfaces`, unless the playbook installs and uses netplan or systemd-networkd on Debian too.
  - ZFS: Debian needs `contrib` enabled plus `zfs-dkms` and `linux-headers-amd64`. Ubuntu needs only `zfsutils-linux`.
  - Time sync: chrony (Ubuntu 26.04 default) vs systemd-timesyncd (Debian default **[unverified]**).
  - Unattended upgrades: present and enabled by default on Ubuntu. Must be installed and configured explicitly on Debian.
  - Ubuntu-only cleanup if wanted: snapd, cloud-init, Pro messaging hooks.
- **Shared behaviour:**
  - APT: both use deb822 `.sources` and `apt-key` is gone (removed in Ubuntu 26.04, deprecated in Debian), so `ansible.builtin.deb822_repository` works for both.
  - Docker repo URL differs only by `/linux/debian` vs `/linux/ubuntu` and the codename.
  - Modules such as `apt`, `systemd_service` and `user` behave the same.
- **rust-coreutils:** Ansible modules call coreutils indirectly (e.g. `command`/`shell` tasks, `stat`-like shell-outs). Behaviour differences of uutils are a risk only for raw shell tasks **[unverified, no Ansible-specific report found]**.

## Open / unverified

- The exact date 24.04 → 26.04 upgrades were enabled (secondary sources say 2026-09-29; `meta-release-lts` confirms they are enabled now).
- Ubuntu Pro free-tier machine count and whether `apt_news=false` silences the esm-apps hook message.
- Debian no-desktop install network backend (ifupdown inferred from netcfg; the wiki says systemd-networkd).
- Security support status of Debian `contrib` packages in general.
- Whether removing snapd or cloud-init from Ubuntu Server affects later `do-release-upgrade` runs.
- Comparative time-to-patch data beyond the single CVE-2024-6387 data point.
