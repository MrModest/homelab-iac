# ZFS modules, safe disk identity and OS OpenZFS versions

Research for issue #5. Facts and trade-offs only, no decisions. Gathered 2026-10-23.
Items marked **[unverified]** were not confirmed against a primary source.

## 1. Ansible modules (`community.general`)

Latest release at time of writing: **community.general 13.5.0** (2026-10-05). Source read from `main` and the `13.5.0` tag.

### `community.general.zpool`

- Exists since **11.0.0** (released 2025-06-16). Author @tomhesse. [source](https://github.com/ansible-collections/community.general/blob/13.5.0/plugins/modules/zpool.py)
- Options: `name`, `state` (`present`/`absent`), `vdevs` (list of `{role, type, disks}`; `type` in stripe/mirror/raidz/raidz1-3; `role` in log/cache/spare/dedup/special), `pool_properties` (`-o`), `filesystem_properties` (`-O`), `mountpoint`, `altroot`, `temp_name`, `disable_new_features` (`-d`), `force` (`-f`).
- Operations: create, destroy, `zpool add` (new vdevs), `zpool remove` (vdevs whose disks are no longer listed), `zpool set` / `zfs set` for properties that differ.
- **Existing pool:** detected via `zpool list <name>` exit code. If the pool exists, it is never re-created. The 13.5.0 docs (not 11.0.0) say: if `vdevs` is omitted on an existing pool, the layout is left unchanged (only properties are reconciled).
- **Layout changes are acted on, not ignored.** When `vdevs` is given and the pool exists, the module parses `zpool status -P -L`, compares normalized layouts, then runs `zpool add` for missing vdevs and `zpool remove` for vdevs containing disks no longer listed. It does not convert topologies (e.g. stripe to mirror via attach).
- **check mode: partial.** In check mode `zpool create`/`add`/`remove` run with `-n` and the dry-run output goes into the diff; property sets are skipped but reported as changes. diff mode: full.
- Known defects (all source-confirmed, issues linked):
  - **`/dev/disk/by-id` paths break idempotence** — open [#10744](https://github.com/ansible-collections/community.general/issues/10744) (2025-08-27). The current layout is read with `zpool status -P -L` (`-L` resolves symlinks to `/dev/sdX`), then partition suffixes are stripped; the requested by-id paths are compared unresolved. Result: on the 2nd run the module thinks the vdev is missing and runs `zpool add ... /dev/disk/by-id/...`, which fails ("is in use"). The fix PR #11020 was closed unmerged (author inactivity).
  - **Not idempotent in general** — open [#10771](https://github.com/ansible-collections/community.general/issues/10771) (2025-08-31): a stripe vdev listing several disks is read back as one stripe vdev per disk, so the diff never matches and the 2nd run fails in `zpool add -n` with zpool's own "use '-f' to override" error (a commenter: "The `-f` suggestion is terrifying"). Also: create-only `filesystem_properties` such as `encryption` fail on the 2nd run because the module tries `zfs set` on them.
  - `special`/`dedup` vdevs not read back — [#12744](https://github.com/ansible-collections/community.general/issues/12744), closed 2026-09-28; fix present in the 13.5.0 source.
  - Danger note (inference from source, **[unverified by test]**): `remove_vdevs()` runs `zpool remove` on any current vdev whose disk is not in the requested set. It runs after `add_vdevs()`, so today a by-id mismatch fails at the `add` step first; if `add` ever succeeded/was skipped, a path-spelling mismatch would target existing vdevs for removal.
- Integration tests never re-declare a pool or assert `not changed` (stated in #12744).

### `community.general.zfs`

- Manages datasets, volumes, snapshots, clones. Options in 13.5.0: `name`, `state` (required), `origin`, `extra_zfs_properties`. [source](https://github.com/ansible-collections/community.general/blob/13.5.0/plugins/modules/zfs.py)
- Idempotence: existence via `zfs list -t all <name>`; existing dataset → only properties with `local`/`received`/`-` source are compared and `zfs set` when different. Create uses `zfs create -p` (parents created).
- `state: absent` runs `zfs destroy -R` (recursive, including dependent clones).
- check mode: partial ("may report changed when it would not", e.g. with pool `altroot`; in 13.5.0 also when sizes are written like `1M` instead of byte counts).
- Unreleased on `main` (version_added **14.0.0**): create-only properties (`encryption`, `keyformat`, `casesensitivity`, `normalization`, `pbkdf2iters`, `utf8only`, `volblocksize`) are no longer `zfs set` on existing datasets; new `create_only_mismatch: fail|warn|ignore` (default `fail`); human-readable size values compared as bytes. Not in 13.5.0.
- `zpool_facts` and `zfs_facts` modules also exist (read-only facts).

## 2. Safe disk identity

- **Naming** — OpenZFS FAQ ["Selecting /dev/ names when creating a pool"](https://openzfs.github.io/openzfs-docs/Project%20and%20Community/FAQ.html): `/dev/sdX` "should only be used ... for development/test pools" (not persistent); `/dev/disk/by-id/` "Best for small pools (less than 10 disks)"; `by-path` for >10 disks; `by-vdev` (`/etc/zfs/vdev_id.conf`) for maximum control; `by-uuid`/`by-partuuid` not recommended.
- **Whole disks** — [zpoolconcepts(7)](https://openzfs.github.io/openzfs-docs/man/master/7/zpoolconcepts.7.html): "recommended mode of operation is to use whole disks"; "When given a whole disk, ZFS automatically labels the disk, if necessary." (On Linux this produces partitions 1 and 9, which is why `zpool status -P -L` shows `/dev/sdX1`. **[unverified]** layout detail, not stated in the man page.)
- **What `zpool create` already refuses** — [zpool-create(8)](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-create.8.html): it "attempts to verify that each device specified is accessible and not currently in use"; mounted devices cannot be overridden; "Other uses, such as having a preexisting UFS file system, can be overridden with `-f`." `-n` shows the config without creating. So without `-f`, existing filesystems/labels block creation; with `-f` (the zpool module's `force: true`) they don't.
- **Detection tools (read-only):**
  - `wipefs -n <dev>` / `wipefs --no-act`: "everything ... except for the write(2) call". `wipefs <dev>` with no options "lists all visible filesystems" and signature offsets; since v2.31 lists all offsets for multi-signature formats (FAT, ZFS, GPT). Supports `-J` (JSON) and `-O` columns; man page advises explicit columns for scripts. (util-linux 2.39.3 man page.)
  - `blkid -p <dev>`: low-level probe bypassing the cache; also reports partition table type (`PTTYPE`). Exit codes: 0 = something identified, **2 = nothing identified**, 4 = usage error, **8 = ambivalent probe result**. ZFS labels show as `TYPE="zfs_member"` ([libblkid zfs.c](https://github.com/util-linux/util-linux/blob/master/libblkid/src/superblocks/zfs.c)).
  - `lsblk -f` / `-J`: FSTYPE/LABEL/UUID per device; man page defers to blkid as authoritative.
  - `zpool import` (no args): "Lists pools available to import"; `-d dir` limits search; `-D` lists destroyed pools only ([zpool-import(8)](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-import.8.html)).
  - `zpool labelclear` is the destructive counterpart ("cannot be undone by ZFS tooling") — not a detection tool.
- Trade-off: `blkid -p` exit code 2 on the whole disk and every partition is the simplest "blank disk" check; `wipefs -n -J` gives a machine-readable list of every signature offset (catches backup GPT / ZFS labels at end of disk). `zpool import` only finds ZFS, nothing else.

## 3. OpenZFS on candidate OSes

Hatchdoor needs OpenZFS ≥ 2.2.

| OS | Release date | OpenZFS (userspace) | Kernel module | Kernel | Fits ≥2.2 |
|---|---|---|---|---|---|
| Debian 12 bookworm | — | 2.1.11-1+deb12u1 (contrib); 2.3.2-2~bpo12+2 (backports) | DKMS | 6.1 | only via backports |
| **Debian 13 trixie** | 13.0: 2025-08-09; 13.7: 2026-09-12 | released with 2.3.2; now **2.3.9-0+deb13u1** (contrib, via trixie-security, changelog 2026-08-23); backports 2.4.4-1~bpo13+1 | **`zfs-dkms`** (contrib), built locally | 6.12.x (stable); backports up to 7.2.x | yes |
| **Ubuntu 24.04 noble** | 2024-04-25 | **2.2.2-0ubuntu9.5** (main) | prebuilt in kernel packages | GA 6.8.0; HWE 7.0.0 | yes |
| **Ubuntu 26.04 resolute** | 2026-04-23 | **2.4.1-1ubuntu5.1** (main) | prebuilt, `linux-main-modules-zfs-<ver>` | 7.0.0 | yes |

Sources: [Debian madison](https://qa.debian.org/madison.php?package=zfs-linux), [Ubuntu madison](https://people.canonical.com/~ubuntu-archive/madison.cgi?package=zfs-linux), Launchpad API (series dates, publish dates), [Debian changelog](https://metadata.ftp-master.debian.org/changelogs/contrib/z/zfs-linux/zfs-linux_2.3.9-0+deb13u1_changelog), [debian.org/releases/trixie](https://www.debian.org/releases/trixie/), `archive.ubuntu.com` Packages indexes.

OpenZFS upstream kernel ranges (`META`, `Linux-Minimum`–`Linux-Maximum`): 2.2.2: 3.10–6.6 · 2.3.2: 4.18–6.14 · **2.3.9: 4.18–7.2** · 2.4.1: 4.18–6.19 · 2.4.4: 4.18–7.2. Latest upstream releases 2026-08-21: 2.4.4, 2.3.9, 2.2.11.

### Debian 13 install path and kernel-upgrade risk

- Packages in **contrib**: `zfs-dkms` + `zfsutils-linux` + `linux-headers-$(dpkg --print-architecture)` ([OpenZFS Debian guide](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/index.html)). Guide suggests trixie-backports (pin 990) for newer ZFS and `DEBIAN_FRONTEND=noninteractive` to avoid a hanging debconf prompt on bad consoles.
- `zfs-dkms 2.3.9-0+deb13u1`: `dkms.conf` has `AUTOINSTALL="yes"`, `BUILD_EXCLUSIVE_KERNEL_MIN="4.18"`, no kernel max; `Recommends: linux-libc-dev (<< 7.3~)`. Only debconf template is a 32-bit-kernel abort question (no license prompt in this version).
- Risk: module is compiled on each kernel install. If the build fails (missing headers, kernel newer than OpenZFS supports), the new kernel boots without ZFS and pools don't import. Stable 6.12 is well inside 2.3.9's range; trixie-backports kernels have already reached 7.2 (= 2.3.9's max), so mixing a backports kernel with stable ZFS can outrun it.
- Secure Boot: DKMS-built modules need a locally enrolled MOK key **[unverified, general knowledge]**.

### Ubuntu install path and risks

- `zfsutils-linux` in **main**; module ships signed with the kernel, no DKMS. Confirmed locally on 24.04.4 GA: `modinfo zfs` → `2.2.2-0ubuntu9.5`, file `/lib/modules/6.8.0-147-generic/kernel/zfs/zfs.ko.zst`, owned by `linux-modules-6.8.0-147-generic`.
- From the 7.0 kernels the module moved to a separate package `linux-main-modules-zfs-<kver>-generic` (Provides `zfs-modules`, `zfs-dkms`), pulled in by `linux-image-generic` (26.04) and `linux-image-generic-hwe-24.04`.
- **24.04 + HWE kernel = userspace/module mismatch.** The noble HWE `linux-main-modules-zfs-7.0.0-38-generic` (7.0.0-38.38~24.04.4) contains `zfs.ko` version **2.4.1-1ubuntu5.1** (read with `modinfo` from the downloaded .deb), while noble userspace stays 2.2.2. GA 6.8 kernel keeps 2.2.2 matched. Whether 2.2.2 tools on a 2.4.1 module are supported by Canonical: **[unverified]**.
- Kernel-upgrade risk on Ubuntu is lower than DKMS (module is built and tested with the kernel), but the module version can jump with an HWE kernel.

### Pool feature portability

- `zfsutils-linux` ships `/usr/share/zfs/compatibility.d/` with `openzfs-2.2-linux`, `openzfs-2.3-linux`, `ubuntu-22.04`, `grub2`, etc. (listed from the Debian 2.3.9 package). `zpool create -o compatibility=<set>` limits enabled features so a pool stays importable on older OpenZFS (e.g. a 2.4-created pool on a 2.2 system).

## 4. Scrub / trim / snapshot tooling

- **Shipped by `zfsutils-linux`** (identical in Debian 2.3.9, Ubuntu 2.2.2 and 2.4.1 packages, read from the .debs):
  - `/etc/cron.d/zfsutils-linux` — **active by default**: TRIM 1st Sunday, **scrub 2nd Sunday of every month** at 00:24, via `/usr/lib/zfs-linux/{trim,scrub}`. The scrub script scrubs every `ONLINE` pool not already scrubbing; opt-out per pool with user property `org.debian:periodic-scrub=disable` on the root dataset (`auto`/`enable` = scrub).
  - systemd templates `zfs-scrub-{weekly,monthly}@.timer` + `zfs-scrub@.service` and `zfs-trim-{weekly,monthly}@.timer` + `zfs-trim@.service` — **not active until enabled per pool** (`systemctl enable zfs-scrub-monthly@<pool>.timer`). Monthly timer: `OnCalendar=monthly`, `Persistent=true`, `RandomizedDelaySec=1h`. Enabling a timer *and* keeping the cron job would scrub twice.
- **sanoid/syncoid**: packaged as `sanoid` — Debian trixie 2.2.0-2 (main), Ubuntu noble 2.2.0-1, resolute 2.3.0-1 (universe). Upstream latest v2.3.0 (2025-06-11). Debian package ships `sanoid.timer`, `sanoid.service`, `sanoid-prune.service`, `/etc/cron.d/sanoid`, binaries `sanoid`, `syncoid`, `findoid`; config is only an example (`/usr/share/doc/sanoid/examples/sanoid.conf`), so `/etc/sanoid/sanoid.conf` must be provided.
- **zfs-auto-snapshot**: 1.2.4-2 in trixie (contrib), noble/resolute (universe). Ships cron jobs in `cron.d`, hourly, daily, weekly, monthly. Upstream ([zfsonlinux/zfs-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot)) last commit 2019-09-30, last tag 1.2.4: effectively unmaintained, snapshot-only (no replication).
- **zfsnap** 1.11.1 also packaged (trixie contrib, Ubuntu universe).

## Open / unverified

- Canonical support stance for 2.2.2 userspace with the 2.4.1 HWE module on 24.04.
- Exact on-disk partition layout ZFS writes for whole disks on Linux.
- Debian Secure Boot + DKMS MOK flow for `zfs-dkms`.
- Behaviour of `zpool` module `remove_vdevs()` with mismatched paths was read from source, not tested.
