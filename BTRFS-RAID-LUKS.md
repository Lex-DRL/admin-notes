# GUI-Arch linux (CachyOS/Manjaro) on BTRFS + RAID + LUKS (2026)

<details>
<summary>▶️ Goal in detail ◀️</summary>

- Getting an Arch-based linux
- installed on BTRFS with subvolumes
- which spans over **multiple SSDs** in BTRFS-RAID1/RAID1cN mode for reliability...
  - https://btrfs.readthedocs.io/en/latest/mkfs.btrfs.html#man-mkfs-profiles
  - SSDs don't need to be the top-spec ones, but they must be **multiple** physical devices: the system partition doesn't need to be top-speed, but it **does** need to be as reliable as possible.
  - So, old SATA SSDs are entirely OK for the OS-partitions-pool, as long as they're not already on the death-bed and are at least somewhat-healthy. The more, the better.
  - For games/downloads/videos/whatever other big-but-less-important files, a higher-speed SSD can be mounted, with no RAID.
- ... each partition - on LUKS2
- decryptable on boot
- with the same password.
</details>

<details>
<summary>▶️ Why? ◀️</summary>

> ### Why?
> 
> #### BTRFS as FS
> It's a conceptual sibling to ZFS:
> - **Checksums**: BTRFS is a checksummed FS, all the way through. Traditional journaling FSes (ext3/4, NTFS on Windows) rely on the journal of operations (and thus can miss quiet data corruption). BTRFS doesn't blindly trust the journal for data integrity, and instead writes checksums next to the actual saved data, **and checks this data against checksum** at every read. For every file. And every piece of metadata (e.g., directory structure).
> - **CoW/snapshots/subvolumes**: BTRFS natively supports copy-on-write (CoW), which is a way of copying files without actually taking extra space (only metadata is copied, but it points to the same underlying FS clusters - and the copy/original actually get their own contents only later, if either is modified - thus, "true" copy happens on **write**). This enables effectively free system snapshots that get created instantly, on a running system, and take no extra space: if the whole system takes 10GiB and only 10MiB were changed between snapshots - the total space used is just around 10.01GiB. Regardless of how many snapshots are there.
> - **Compression**: not only data duplication could be mitigated by CoW, but even the data that gets written could (and should) be compreesed, saving even more space - for almost free in terms of system resources used.
> - **BTRFS-RAID**: regular RAID operates at the lowest level (on bare metal) and thus has no idea of **what** is stored on it. Clusters of data come - RAID syncronizes raw bytes between devices. That's it. So, whenever a new device is added to the array, it must be rewritten with the full copy of data... even if the "data" is merely an empty partition. But BTRFS supports somewhat-like-a-RAID **at its own level**, which **knows** what data it works with. So any RAID-relevant operation is vastly more efficient, and replacing a device is much easier thing than with an oldschool bare-metal RAID.
> 
> #### RAID1 on a consumer PC
> Normally, if data gets corrupt, BTRFS would notice that on the next attempt to read it... but at this point it won't be able to do anything about it except throwing a read error. However, by modern standards, OS partition usually needs neither much space (100GiB is generous) nor ultra-fast speed (even SATA speeds are normally enough) - instead, it needs **reliability**. So, even though RAID came from servers, now - even for consumer PCs - it gets realistic to buy a few small SATA SSDs (or even use old ones that you were about to throw away or could buy dirt-cheep second-hand) and set up BTRFS on multiple physical devices in RAID1 mode.
> 
> Then, if a corruption happens, BTRFS would not only be able to read the corrupt file, but would just self-heal the data the next time it's accessed. Silently. Without an error. Without rebooting the PC into any special test/recovery mode.
> 
> Together, it:
> - reduces e-waste by extending the life of SSDs you would otherwise be afraid to store important data on;
> - completely eliminates the stress of losing data due to power outage or hardware malfunction (if regular scrub is enabled);
> - prolongues the life of your Linux installation, so in a long term it also saves time and energy.
> 
> Win-win, no matter how you look at it. The only downside is - yes - this is (relatively) hard to set up. And the lack of GUI tools in Linux ecosystem doesn't help.
> 
> However...
>
> > #### ⚠️ RAID IS NOT A BACKUP ⚠️
> > 
> > It's a safety net, yes. And a huge one. But, even though it protects data from getting corrupt in daily use, the whole RAID array can still die all at once. E.g., the whole PC getting destroyed or just all the SSDs failing together (highly probable if they were added to the array with about-the-same lifespan left - e.g., same model, added new).
> > 
> > **AFTER** this installation is moved to BTRFS-RAID - ideally, it should also be regularly backed-up in compliance with [3-2-1 rule](https://en.wikipedia.org/wiki/Backup#3-2-1_Backup_Rule).
> 
> #### LUKS
> Encryption in the age of **personal** computers (so, since 2000s, let alone in 2020+) isn't a luxury, it's essential. All your personal data, photos, browser history, keys and passwords to your other devices, as well as **everything** else that you store is freely available to anyone who has a physical acces to your device... unless it's encrypted. So if your laptop gets stolen, or you share your room with a roommate in a student dorm, or you might leave your work-PC unattended when you go for a coffee break, or you just live in a country with oppressive government which loves getting into its citizen's business then jailing them for it - having no encryption is effectively an open doorframe without any door in it. Yes, not even a door without a lock - no door at all. You should similarly treat any data that isn't encrypted: as a flash drive which you've lost in a central square of your city. Or as the same stuff printed and left in a room with no door in a public place.
> 
> I.e., full encryption on personal devices should've become an absolutely must-have standard thing long ago, but somehow it still haven't in 2026. You **should** have it - always and on every your device. No exceptions. Like you should have a door with a lock on your apartment. This isn't a paranoia - it's just common sense.
> 
> LUKS is the best way to have encryption on Linux.
</details>

> [!WARNING]
> Terminal is unavoidable. Despite each of the components being widely recommended as "best practice", no GUI tool can do them combined together.

---

## 0. Preliminary work

Throughout the guide, it's helpful to review the state of all partitions with:

```bash
clear && lsblk -o NAME,PARTLABEL,LABEL,SIZE,MOUNTPOINTS,FSTYPE,FSVER,UUID,MODEL
```

### 0.1. Planning ahead

The whole process is done like this:
1. Let the official installer do its job and [set up a close-enough intermediate installation on a temporary SSD](#1-initial-tempintermediate-installation);
2. [Prepare the properly-formatted LUKS-encrypted main BTRFS partition](#2-preparing-partition-on-main-ssd) for this Arch to live on;
3. [Move the temp install to it](#3-clone-temp-install-to-the-main-ssd), updating all UUIDs;
4. [Clear the temp SSD after confirming the main SSD still boots](#4-clean-up-the-intermediate-ssd);
5. [Repeat the LUKS-setup process for additional SSDs + expand the main BTRFS onto them](#5-expand-btrfs-to-additional-ssds-for-raid), turning them to a BTRFS-RAID1-array.

> [!WARNING]
> Multiple spare SSDs are assumed for the final setup.

To save yourself from many headaches, the OS partitions must have the same size.
- So, if SSDs have different sizes, the smaller one is used as the OS-partition cap, ...
- ... and **over-provisioning** (leaving an unpartitioned space at the end of the SSD) is **HIGHLY** recommended for balancing the load on the SSD wear.
  - Recommended: 10-20% of total SSD size, 15% as a reasonable mid-point (extends the SSD lifespan **A LOT**, while still providing the most of the usable space).
  - AFAIK, all Samsung SSDs auto-use all unallocated space for over-provisioning, other brands need double-checking.
- So, if 3 SSDs are available - 512GiB + 256GiB + 128GiB - their **shared** size is 128GiB. Minus 15% for over-provisioning on the smallest one, so `128*0.85=108.8` GiB, minus EFI partition (up to 2GiB, see below) on each (better have it on every SSD as a backup) - so the system partition is about 100-105GiB.
- The remaining space on the other SSDs (still respecting over-provisioning on each) can be used for other partitions - SWAP or an additional BTRFS with lesser or even no redundancy. E.g., with 3 SSDs from above:
  - Each SSD has EFI partition. Then...
  - the main `/` spans over all 3 SSDs, as 100GiB partitions - in `raid1c3` mode
  - an additional BTRFS pool with `@home` subvolume (mounted as `/home`) uses another ~100GiB on the thirst two SSDs - in `raid1` mode
  - Additionally, there's a SWAP, plus a 5th partition in the remaining space of first SSD for shared `/downloads` - no redundancy at all, just the default BTRFS' `dup` for metadata/system, `single` for the actual data.
- Since bigger SSDs are usually newer (and thus, more reliable), the drives should be connected in the size-descending order.

The first/best SSD is gonna be used for mounting as a "main" one (even though devices in a BTRFS-array are "equal" siblings, they still have their IDs under the hood).

And initial installation should be performed on the last SSD.

### 0.2. SSD secure erase

https://wiki.archlinux.org/title/Solid_state_drive/Memory_cell_clearing

To restore out-of-the-box speeds on SSDs, it's worth to secure-erase them **AFTER ALL RELEVANT DATA IS ALREADY BACKED UP TO A DIFFERENT DEVICE**, but before new OS installation.

> [!CAUTION]
> Pay **EXTREME** attention to **WHICH** devices you secure-eraze.
> 
> Ideally, it's worth to shut down PC and physically unplug any SSDs that **WON'T** be cleared - before proceeding to secure erase.

Best choice: `Erase Disk` GUI tool in PartedMagic boot image (also has tools for NVMe + SATA "sanitize").

<details>
<summary>▶️ Terminal: Second best - manual Secure Erase for SATA SSDs with hdparm ◀️</summary>

```bash
ERASED_SATA_DRIVES=(sda sdb sdc)

# check whether device is frozen:
for sdX in "${ERASED_SATA_DRIVES[@]}"; do
  echo "\n\n-----$sdX"
  sudo hdparm -I "/dev/$sdX" | grep frozen
done

# if frozen - to unfreeze, send the PC to sleep:
sudo systemctl suspend

# enable device-level encryption with a 'PasSWorD' password:
for sdX in "${ERASED_SATA_DRIVES[@]}"; do
  echo "\n\n-----$sdX"
  sudo hdparm --user-master u --security-set-pass PasSWorD "/dev/$sdX"
done
# ... and check it's enabled:
for sdX in "${ERASED_SATA_DRIVES[@]}"; do
  echo "\n\n-----$sdX"
  sudo hdparm -I "/dev/$sdX"
done

# the actual secure erase:
for sdX in "${ERASED_SATA_DRIVES[@]}"; do
  echo "\n\n-----$sdX"
  sudo hdparm --user-master u --security-erase PasSWorD "/dev/$sdX"
done
```
</details>


### 0.3. SMART tests

<details>
<summary>▶️ Terminal: smartctl ◀️</summary>

```bash
SMART_DRIVES=(sda sdb sdc)

for sdX in "${SMART_DRIVES[@]}"; do echo "\n\n-----$sdX"; sudo smartctl -c "/dev/$sdX"; done
for sdX in "${SMART_DRIVES[@]}"; do echo "\n\n-----$sdX"; sudo smartctl -l error "/dev/$sdX"; done

for sdX in "${SMART_DRIVES[@]}"; do echo "\n\n-----$sdX"; sudo smartctl -c "/dev/$sdX" | grep -i convey; done
for sdX in "${SMART_DRIVES[@]}"; do echo "\n\n-----$sdX"; sudo smartctl -t conveyance "/dev/$sdX"; done

for sdX in "${SMART_DRIVES[@]}"; do echo "\n\n-----$sdX"; sudo smartctl -t long "/dev/$sdX"; done

clear && for sdX in "${SMART_DRIVES[@]}"; do echo "\n\n-----$sdX"; sudo smartctl -a "/dev/$sdX" | grep "remain"; done
```
</details>

### 0.4. Optimal sector size

> [!IMPORTANT]
> **TL;DR:**
> - For BTRFS-on-LUKS: just **EXPLICITLY** specify `4096` for all drives on all layers - whether SATA or NVMe, SSD or HDD, BTRFS or LUKS.
> - For other FSes (e.g., SWAP-on-LUKS): make it equal to physical sector size... to determine which, read further.

To check it (`PHY-SEC` column):

```bash
lsblk -td
lsblk -o NAME,SIZE,ALIGNMENT,MIN-IO,OPT-IO,PHY-SEC,LOG-SEC,ROTA,SCHED,RQ-SIZE,RA,MODEL
```

<details>
<summary>▶️ Example output ◀️</summary>

```
NAME      SIZE ALIGNMENT MIN-IO OPT-IO PHY-SEC LOG-SEC ROTA SCHED       RQ-SIZE   RA MODEL
sda     476,9G         0    512      0     512     512    0 mq-deadline      64  256 SanDisk SD8SB8U512G1122
sdb     447,1G         0    512      0     512     512    0 mq-deadline      64  256 WDC WDS480G2G0A-00JH30
sdc     119,2G         0    512      0     512     512    0 mq-deadline      64  256 SanDisk SDSSDHP128G
sdd     931,5G         0    512      0     512     512    0 mq-deadline      64  256 Samsung SSD 860 EVO 1TB
sde      12,7T         0   4096      0    4096     512    1 bfq              64 8192 TOSHIBA MG07ACA14TE
zram0     7,4G         0   4096   4096    4096    4096    0                      256 
nvme0n1 931,5G         0  16384 131072     512     512    0 kyber           256  256 Samsung SSD 980 1TB
nvme1n1   1,8T         0    512      0     512     512    0 none           1023  128 Samsung SSD 970 EVO Plus 2TB
nvme2n1   1,8T         0   4096      0    4096    4096    0 none           1023  128 WD_BLACK SN850X 2000GB
```
</details>

[For NVMe](https://wiki.archlinux.org/title/Solid_state_drive/NVMe), there also are these commands (in `nvme-cli` package):
```bash
nvme id-ns -H /dev/nvmeXnY | grep -i lba
nvme id-ctrl /dev/nvmeXnY | grep -i awupf
```

<details>
<summary>▶️ Example output ◀️</summary>

```
LBA Format  0 : Metadata Size: 0   -  Data Size: 512  - Relative Performance: 0x2 (Good)
LBA Format  1 : Metadata Size: 0   -  Data Size: 4096 - Relative Performance: 0x1 (Better)
```
</details>

... to be later formatted with `nvme format /dev/nvmeXnY -l <lba_format_index>` - but again, the drive's preference is irrelevant because...

<details>
<summary>▶️ Explanation - A LOT of details ◀️</summary>

> **Theoretically**, we should use the same **physical** sector size throughought the entire stack (https://wiki.archlinux.org/title/Advanced_Format) - so, for both LUKS and BTRFS (and if LVM was there - for it, too). But in practice, this topic is a total mess...
> 
> #### HDD
> In a vacuum, for HDDs - we actually **should** do as described above. Some (old) HDDs report 512 bytes, some report 4K. And some - 512 logical sector / 4K physical, aka "512e". Physical is always the king.
> 
> #### SSD: SATA vs NVMe
> - SATA SSDs (almost?) always report 512-byte physical/logical.
> - Some NVMe SSDs do the same (my Samsung 980 1TB), some report 4k/4k (WD_BLACK SN850X 2000GB)
> 
> ... but in both cases this means nothing, because DRAM cache does write/erase operations in bulk anyway - so `PHY-SEC` column from `lsblk` output isn't tied to the NAND page size, `OPT-IO`/`MIN-IO` columns are. But their size is usually 16K-128K, way above what LUKS supports... and often it's not even reported at all (0), or simply falls back to `PHY-SEC` value.
> 
> ... but some cheapest noname-brand SSDs have no DRAM, so they **do indeed write** immediately, in the NAND page sizes
> 
> ... but in such cases, there's no guarantee whether any of `PHY-SEC`/`OPT-IO`/`MIN-IO` matches the NAND page size.
> 
> So, there's no reliable way to run a command and detect an optimal sector size for your specific SSD - each model requires a whole investigation online.
> 
> #### BTRFS sector size
> https://btrfs.readthedocs.io/en/latest/Subpage.html
> 
> Basically, for `x86_64`, the size is hardcoded to always be 4K (`4096`), because it's locked on the kernel's page size. To check:
> ```bash
> getconf PAGESIZE
> ```
> 
> Even if an SSD reports 512-bytes physical sectors, and we format both LUKS and BTRFS on it **without** specifying a sector size (so, it defaults to the physical one - CachyOS' installer does exactly that) - `lsblk` will tell that BTRFS has the same 512-byte size... but **ACTUALLY**, it doesn't.
> To truly check the **real** sector size, we need to read the BTRFS superblock:
> ```bash
> sudo btrfs inspect-internal dump-super /dev/mapper/<decrypted-device> | grep -i sectorsize
> ```
> 
> And it probably will be 4K, regardless of which `--sectorsize` value was given to / detected by `mkfs.btrfs`.
> 
> #### LUKS sector size
> It must be in 512-4K range. But BTRFS on top of it would have 4K anyway, so - even if the underlying device "prefers" 512 - BTRFS would still send data in 4K chunks which would be just split into 512-byte ones, and each would be encrypted individually... just to be reassembled into even bigger (16K-128K) chunks at the hardware level. I.e., `8x` overhead at LUKS level, for no benefit.
> 
> ...but if we don't specify LUKS sector size explicitly, it will default to the reported physical one, which (for SSDs) likely has nothing to do with the actual physical granularity of flash cells (see above). So we should explicitly tell LUKS to use 4096 - i.e., the only overlap with BTRFS.
> 
> #### The only exception
> - If the FS **IS NOT** BTRFS (i.e., SWAP),
> - and it supports other sizes,
> - and it's not on an LVM which contains another BTRFS volume,
> - and the drive's reported sector size is different from 4K (by `nvme id-ns` for NVMe, `lsblk -td` for SATA),
> - and it's in 512-4096 range,
> - and none of the drive's partitions could "extend" to any other device (e.g., with LVM) which **would have** 4K sectors
> 
> ... then **and only then** it could be formatted with a different value (probably, 512).
> 
> #### 4K BTRFS on 512 SSD - the only drawback
> On sudden shutdown during write, the SSD might write **some** of the 512-byte chunks that compose a single 4K sector, so it will end up partially corrupt. But:
> - See above about DRAM cache.
> - The written file probably will be corrupt anyway: sector more, sector less - who cares **when** it was interrupted? It would matter only for the extremely rare edge case of power loss **EXACTLY** while writing the very last sector of a file.
> - Our setup is **specifically** designed around BTRFS-RAID1, which would auto-heal it on the next scrub/read, if there is anything to save.
> 
> #### Test it
> 
> None of the terminal commands guarantee that the reported size is indeed the size that's "desired" by your SSD. But if you want to check - in addition to `lsblk` and `nvme id-ns`, there are:
> ```bash
> smartctl -a /dev/sdX | grep -i phys
> for sdX in sda sdb sdc sdd sde nvme0n1; do
>   echo "---------- $sdX ----------"
>   sudo smartctl -a "/dev/$sdX" | grep -i phys
> done
> 
> # Via kernel / sysfs entries:
> cat /sys/class/block/sdX/queue/physical_block_size
> cat /sys/class/block/sdX/queue/logical_block_size
> for sdX in sda sdb sdc sdd sde nvme0n1; do
>   echo "---------- $sdX ----------"
>   sudo cat "/sys/class/block/$sdX/queue/physical_block_size"
> done
> 
> # SATA-only
> hdparm -I /dev/sdX | grep -i phys
> for sdX in sda sdb sdc sdd sde; do
> echo "---------- $sdX ----------"
>   sudo hdparm -I "/dev/$sdX" | grep -i phys
> done
> ```
</details>

### 0.5. Optimal `--iter-time` for LUKS-format
https://wiki.archlinux.org/title/Dm-crypt/Device_encryption

**TL;DR:**
- Estimate how much the top modern flagship PC is more powerful than your PC's CPU/RAM. Use some test results from public multi-threaded benchmarks like CineBench: `top score / your score`.
- Multiply it by `2000`. This is your value for `--iter-time`. It roughly equals the time (in ms) that each unlock attempt would take.
- If your PC is drastically slower (so the calculated `--iter-time` is over 10k), you can reduce it by extending password:
  - Min password length (with the raw calculated iter-time): 8-10 **elements** (points of entropy)
  - A password element is:
    - 1 character **IF THE PASSWORD IS RANDOMLY GENERATED** from entire alphanumeric set (lowercase a-z, uppercase A-Z, 0-9, contains each type) - not "looking random", not a "secret generation pattern only I know", **ACTUALLY** random;
    - 1 word in a passphrase.
  - Each extra element over 8 allows you to divide `--iter-time` by ~10, but it should never go below 1000.
  - 8 points of entropy as a base (with 2k iter-time **on a top-tier PC**) is just "absolute minimum", nothing more - and only if you **AREN'T** a potentially valuable victim for targeted attacks. Actually good passwords start at 15 points of entropy, and for future-proofing - 20+ is a standard.

<details>
<summary>▶️ Details ◀️</summary>

> The `--iter-time` value needs to be balanced: too small - and it could be brute-forced. Too big - and each decryption would take very long time (when used with GRUB, even slower - to an ATROCIOUS extent).
> 
> On modern multi-core CPUs, 2k (`2000`, aka 2s) or even 5k-10k is a reasonable starting point. But this is for the **TOP-TIER MODERN PCs**. Keep in mind:
> - `--iter-time` indirectly specifies the chosen number of iterations to encrypt with. Then, this number of iterations is performed during each unlock of the device. But it's calculated **on your particular PC**, and anlock attempt with the same number of iterations can be done on **ANY** PC (or even a cluster of them). Encryption with `--iter-time 5000` would mean the decryption would take about 5s on your PC... **BUT**, if this exact SSD is gonna be decrypted on another PC, twice more powerful - it would take twice less time. And if it's a server farm thousands of times more powerful than your PC - it will literally be thousands of times faster. With significant performance disparity and a short password, it can actually be brute-forced.
> - More powerful CPUs than yours would be proportionally faster to decrypt. If your CPU is orders of magnitude slower than the current top-tier ones (quite realistic for a NAS PC with a DDR3-era 2/4-core CPU vs DDR5-era flagship Ryzen), you should proportionally increase iter-time.
> - If such increase makes the decryption unreasonably slow on your PC, a longer password is the only way.
> - LUKS supports multiple keyslots. For a low-spec NAS-PC (which is gonna be normally decrypted over LAN), you could:
>   - Do the initial encryption (to take the first/main keyslot) with a temporary short password, insufficient for the chosen iter-time - but comfortable to unlock during this setup;
>   - Then set up a proper "recovery" password **in a second keyslot**: normal length with appropriate iter-time (insufferably slow on your PC) - to be used only for maintenance when you'll need to manually type it from the physical keyboard;
>   - And after the entire setup is finished and remote-unlock-over-LAN (on boot) is also configured, you replace the first keyslot with the same fast iter-time, but **ridiculously long password** - 128+ random characters, which you will normally just paste from your password manager over ssh from another PC, and never type manually;
>   - You're using a secure password manager in 2020+, **don't you?** (e.g., KeePassXC)
> 
> To test  the actual decryption speed:
> ```bash
> ITER_TIME=2000
> 
> TMP_IMG="/tmp/luks-test-time.img"
> TMP_KEYFILE="/tmp/luks-test-dummy-key"
> 
> # Create a small throwaway test file - no real partition
> sudo fallocate -l 64M "$TMP_IMG"
> # ... and a dummy keyfile
> echo "dummy_but_long_enough_TESTING_password69" | sudo tee "$TMP_KEYFILE"
> 
> # Format it with the candidate iter-time, on THIS machine
> sudo cryptsetup luksFormat \
>   --type luks2 \
>   --cipher 'aes-xts-plain64' \
>   -h sha256 \
>   -s 512 \
>   --pbkdf argon2id \
>   --sector-size 4096 \
>   --iter-time $ITER_TIME \
>   --use-urandom -v \
>   --batch-mode --key-file "$TMP_KEYFILE" \
>   "$TMP_IMG"
> 
> # Measure the actual unlock time
> time sudo cryptsetup open \
>   --type luks2 \
>   --key-file "$TMP_KEYFILE" \
>   "$TMP_IMG" tmp-luks-test
> sudo cryptsetup close tmp-luks-test
> 
> # Inspect what parameters it actually landed on
> sudo cryptsetup luksDump --type luks2 "$TMP_IMG"
> sudo rm "$TMP_IMG"
> sudo rm "$TMP_KEYFILE"
> ```
</details>

---

## 1. Initial (temp/intermediate) installation

Install with a graphical installer (Calamares?), but:
- On a **temp** (last/smallest) SSD
- Choosing the "wipe whole disk and install" option
- With BTRFS chosen
- And encryption enabled (type the same password as you want for LUKS in the end)

The last two - so that the installer would properly configure mkinitcpio hooks.

### 1.1 Bootloader choice
`systemd-boot` is preferred:
- Easier configuration (just directly tweaking the boot config files, no `grub-update` convolution);
- **Uses the real linux to decrypt on boot** - this means the same decryption speed as in running Arch, NOT a much slower one as with GRUB;
- The password-prompt screen is just nicer (graphical);
- Since initcpio is on ESP - `systemd-boot` automatically (out of the box on CachyOS) decrypts multiple devices with the same shared password...
- ... and later, `tinyssh`/`dropbear` could be relatively-easily set up to decrypt over LAN (pre-boot). Crucial for a NAS.

But **downsides** of `systemd-boot` (against GRUB):
- Since kernels/initcpio are on ESP: if an attacker has physical access to the PC, they can modify initcpio to steal passwords on next boot. Can be mitigated with boot-files verification or TPM2, but it's a whole another subject.
- No snapshot entries in boot menu. Could be somewhat mitigated with simply having "last successfull boot" entry that points to a "stable" snapshot. Needs some (small) scripting to update the snapshot/link after successful boot.
- Any contents of the `/boot` itself (i.e., kernels) won't be a part of snapshot... though, a copy could be. Some `rsync` scripting required.

After setup is complete - reboot to the installed Linux and do the rest from it.

---

## 2. Preparing partition on main SSD

### 2.1. Partitioning

Using graphical `KDE Partition Manager` / `Gparted`:
- When creating partitions, **1MiB alignment must be enabled** (✅ at the very bottom of partition-creation dialog) - crucial for SSDs.
- GPT
- First partition - ESP (EFI System Partition):
  - FS: fat32
  - Partition/FS label: `EFI` or `ESP`
  - Must have `boot` flag (`bios-boot` also won't hurt)
  - Size:
    - **systemd-boot** style: `2GiB` - if it's mounted as `/boot`, so it also contains kernels.
    - **GRUB** style: `128-256MiB` - if `/boot` with kernels is elsewhere (i.e., part of main partition), and EFI only contains bootloader itself.
- Second partition - for `/`.

### 2.2. LUKS on main partition

#### LUKS format
```bash
sdX=sdXi
ITER_TIME=2000
sudo cryptsetup luksFormat \
  --type luks2 \
  --cipher 'aes-xts-plain64' \
  -h sha256 \
  -s 512 \
  --pbkdf argon2id \
  --sector-size 4096 \
  --iter-time $ITER_TIME \
  --use-urandom -v \
  -y \
  /dev/$sdX
```

> [!TIP]
> Even with the "Verify passphrase" prompt, it's possible to mistype password (twice). To prevent it, it's highly recommended to type it with different speeds on each request:
> 1. As fast as you normally would type it when unlocking.
> 2. Very carefully, while looking at keyboard and clearly pressing every key - like boomers do.

<details>
<summary>▶️ Argument values compared against best practices by LLM (Claude 5, 2026) ◀️</summary>

> - `--cipher aes-xts-plain64` — still the current standard, confirmed directly from RHEL 10's own documentation and a March 2026 LUKS2 deep-dive: it's the default for a reason, and it's hardware-accelerated on your CPU (Gemini Lake has AES-NI). No reason to change it.
> - `-h sha256` — here's a nuance worth knowing: with `--pbkdf argon2id` (which you're using), this hash parameter mostly stops being the thing standing between an attacker and your data — Argon2id is a self-contained construction, not built on top of an external hash the way PBKDF2 is. What `--hash` still controls is the LUKS2 header's own digest/verification mechanism and the anti-forensic (AF) splitter — a real but secondary role. SHA-256 is unbroken and entirely adequate there; no reason to bump to SHA-512.
> - `-s 512` — worth clarifying what this number actually means, since it trips people up: XTS mode uses *two* internal keys (one for encryption, one for the "tweak" that makes each block's encryption unique), so a "512-bit" XTS key is really AES-256 used twice — not some exotic 512-bit AES that doesn't exist. `-s 512` with `aes-xts-plain64` is exactly the current cryptsetup default and gives you genuine AES-256 protection. Correct as-is.
> - `--pbkdf argon2id` — still the current best choice, confirmed as the default across every current source I checked (RHEL, Talos, Fedora). Correct.
> - `--use-urandom` — this avoids `/dev/random`'s old (and now largely obsolete) blocking behavior, where it could theoretically stall if the kernel judged its entropy pool "insufficient." Modern kernels have unified the two sources into the same CSPRNG once properly seeded at boot, so this isn't a security downgrade — it just guarantees `luksFormat` won't hang if you run it early in a live-boot session with less available entropy. Reasonable to keep.
</details>

> [!IMPORTANT]
> #### LUKS on SSDs
> https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD)
> 
> LUKS2 (unlike LUKS1) can remember certain decryption flags, and auto-apply them by default the next time. SSDs actually need at least one such flag set (`--allow-discards`) - or you'll have to manually set it on each decrypt. To remember it:
> 
> ```bash
> sdX=sdXi
> DECRYPTED_NAME=main_crypt_A
> sudo cryptsetup open \
>   --type luks2 \
>   --allow-discards \
>   --persistent \
>   /dev/$sdX "$DECRYPTED_NAME"
> sudo cryptsetup close "$DECRYPTED_NAME"
> ```
> 
> To make sure that the `discards` flag is saved:
> ```bash
> sudo cryptsetup luksDump --type luks2 /dev/$sdX
> ```
> 
> Any later unlocks from terminal would be just:
> ```bash
> sdX=sdXi
> DECRYPTED_NAME=main_crypt_A
> sudo cryptsetup open /dev/$sdX "$DECRYPTED_NAME"
> ```
> 
> The following Arch Wiki page also recommends enabling `--perf-no_read_workqueue` and `--perf-no_write_workqueue`: 
> https://wiki.archlinux.org/title/Dm-crypt/Specialties#Disable_workqueue_for_increased_solid_state_drive_(SSD)_performance
> 
> Buuut... in reality, it's not as clear recommendation, as the page puts it.
> 
> <details>
> <summary>⚠️ Important clarification from Claude 5 ⚠️</summary>
> 
> > **`--perf-no_read_workqueue`/`--perf-no_write_workqueue` — real flags, correctly described, but the "should" is doing too much work.** These trace to an actual 2020 kernel patch from Cloudflare engineer Ignat Korchagin, built on real research (his "Speeding up Linux disk encryption" writeup) — the mechanism is genuine: they make dm-crypt process crypto inline on the calling thread instead of handing it to a kernel workqueue thread, which cuts latency on very fast storage where that hand-off overhead is a meaningful fraction of total I/O time. ArchWiki does present this as a general SSD recommendation. But independent testing tells a messier story than "just turn it on":
> > - The Ubuntu kernel developer who reviewed the original patch benchmarked it on bare-metal (Ryzen 7 2700) and found *mixed* results — in one concrete configuration (4M block size, queue depth 16, single job) enabling the flags dropped read throughput from 4773 MiB/s to 2782 MiB/s, a **42% regression**, not an improvement.
> > - A 2026 forum thread from someone on a genuinely fast 2TB Solidigm NVMe reported the flags made performance "a lot worse," across multiple KDiskMark categories.
> > - Someone doing almost exactly your exercise (testing LUKS+NVMe tuning empirically) found enabling both flags via crypttab gave "no noticeable improvement" at all.
> > 
> > So: real flags, real mechanism, real motivating research — but "should be set" overstates it. The honest framing is "worth testing on your actual hardware and workload," not "apply by default."
> > 
> > ---
> > 
> > ##### The line, in Cloudflare's own words
> > Coverage of their original write-up quotes Cloudflare's own conclusion directly: *"In the future, Cloudflare plans to transfer the prepared patches to the main Linux kernel, but before that they will need to be modified, as they are optimized for a specific workload and do not cover all application areas."* That's the people who built this feature stating plainly that it isn't a universal win — it's tuned to a specific shape of I/O. And the contemporary LWN discussion thread around the original post made the same point independently: *"That's what you get with one size fits all... It would be a miracle if all scenarios performed ideally."*
> > 
> > ##### What that specific shape actually is
> > Pulling together the mechanism and the concrete benchmarks, the dividing factor isn't queue depth alone — it's **individual request size, relative to whether dm-crypt would otherwise have split that request across multiple cores to encrypt it in parallel.** The kernel docs describe this split-for-parallelism behavior explicitly as the thing you give up by going inline. So:
> > - **Small-to-moderate individual I/O, where the win you're chasing is per-request latency, not aggregate throughput** — this is where the flags are close to a clean win, because there was no multi-core splitting happening for those requests anyway (they're below the size where splitting kicks in), so bypassing the workqueue only removes overhead with nothing to lose. Cloudflare's own production case fits this precisely: edge cache servers serving individually modest cached objects to enormous numbers of concurrent connections, where tail latency per request was the metric that mattered, not sustained MB/s on any one stream. It's the same reasoning behind the zen-kernel maintainers' interest in this — zen is explicitly a low-latency, desktop-responsiveness-oriented kernel, and their advocacy for the flags was framed around *latency*, with throughput described as a secondary, "even" bonus.
> > - **Large individual requests, where the default's multi-core split-and-parallelize is doing real work** — this is exactly the regime that produced the 42% regression in the benchmark from a couple messages ago: 4MB blocks at queue depth 16. Forcing that inline serializes encryption of what would otherwise be sharded across your CPU's cores, and you lose real aggregate throughput to save a comparatively small amount of per-request scheduling latency.
> > 
> > So the line isn't really "SSD vs. not" or even cleanly "queue depth" by itself — it's **"are you optimizing for the responsiveness of many small-ish operations, or for the sustained throughput of large ones."**
> > 
> > ##### Where that puts your NAS
> > Your dominant traffic — large media files, backups, bulk transfers over SMB/NFS — sits squarely on the large-request, throughput-optimized side of that line, the side with a documented regression, not the side Cloudflare actually validated. And it also explains one data point: BTRFS scrub, the operation that had the documented crash history with these flags, is itself a large-scale, throughput-heavy, whole-filesystem read scan — mechanistically the same category as the 4MB/QD16 case that regressed, not a coincidence. Everything lines up on one side for you: your workload's defining traffic pattern, and the one historical incident tied to your exact stack, both fall on the side of this line where the evidence points against enabling, not toward it. That sharpens last message's answer rather than changing its direction — I'd still leave these off, and now with an actual mechanism behind the "why" instead of just a hardware-class inference.
> </details>

#### Add extra LUKS keyslot

> [!WARNING]
> Keyslot indices are 0-based (first/default keyslot has number 0). So, keyslot 1 is second, 2 is third, etc.

<details>
<summary>▶️ Password ◀️</summary>

```bash
sdX=sdXi
ITER_TIME=2000
KEY_SLOT=1
sudo cryptsetup luksAddKey \
  --type luks2 \
  --keyslot-cipher 'aes-xts-plain64' \
  -h sha256 \
  --keyslot-key-size 512 \
  --pbkdf argon2id \
  --iter-time $ITER_TIME \
  --key-slot $KEY_SLOT \
  -v \
  -y \
  /dev/$sdX
```
</details>

<details>
<summary>▶️ Keyfile ◀️</summary>

```bash
sdX=sdXi
ITER_TIME=2000
KEY_SLOT=1
KEY_PATH=/_crypto_keys/crypto_keyfile.bin

# Generate random keyfile of 4 KiB:
sudo dd bs=512 count=8 if=/dev/urandom of="$KEY_PATH" iflag=fullblock
# Ensure it has the right size:
echo "Keyfile size:" && sudo wc -c "$KEY_PATH"
# Access rights:
sudo chown root:wheel "$KEY_PATH"
sudo chmod 640 "$KEY_PATH"

sudo cryptsetup luksAddKey \
  --type luks2 \
  --keyslot-cipher 'aes-xts-plain64' \
  -h sha256 \
  --keyslot-key-size 512 \
  --pbkdf argon2id \
  --iter-time $ITER_TIME \
  --key-slot $KEY_SLOT \
  -v \
  -y \
  /dev/$sdX "$KEY_PATH"
```
</details>

<details>
<summary>▶️ Clear keyslot before replacing ◀️</summary>

```bash
sdX=sdXi
KEY_SLOT=1
sudo cryptsetup luksKillSlot /dev/$sdX $KEY_SLOT
```
</details>

<details>
<summary>▶️ Make key slot preferred (on decryption, tried first by default) ◀️</summary>

```bash
sdX=sdXi
KEY_SLOT=1
sudo cryptsetup config --type luks2 --key-slot $KEY_SLOT --priority 'prefer' /dev/$sdX
```
</details>

### 2.3. BTRFS on main partition

#### Open LUKS
If the created LUKS isn't left open after [LUKS-discard step](#luks-on-ssds):

```bash
sdX=sdXi
DECRYPTED_NAME=main_crypt_A
sudo cryptsetup open --type luks2 /dev/$sdX "$DECRYPTED_NAME"
```

#### Format BTRFS

```bash
DECRYPTED_NAME=main_crypt_A
LABEL=CachyOS
sudo mkfs.btrfs --sectorsize 4096 --checksum xxhash --features block-group-tree --label "$LABEL" "/dev/mapper/$DECRYPTED_NAME"
```

<details>
<summary>▶️ Arguments explained ◀️</summary>

> - `--sectorsize`: explicitly setting it to `4096` to match the LUKS layer, explained above.
> - `--checksum`: [algorithm to use for checksums](https://wiki.archlinux.org/title/Btrfs#Checksums), default is `crc32c`. But `xxhash` is faster, more reliable and more hardware-compatible at the same time. The only downsides are:
>   - it requires kernel 5.5 and `btrfs-progs` ~5.4/5.5 - so a BTRFS with `xxhash` **MIGHT** become non-mountable on some **TRULY** ancient recovery disks, or might not work with other low-level tools like that (which reimplement their own interaction with BTRFS, instead of relying on kernel/btrfs-progs). As long as we deal with this partition from within a somewhat-up-to-date linux (whether installed or from a live DVD), there's full feature parity with crc32c. And since `systemd-boot` uses the same kernel/driver as installed in the system, it's irrelevant there. Might be relevant for GRUB.
>   - **slightly** bigger footprint of checksums: 4 bytes per block (so, per 4096) on crc32c vs 8 bytes on xxhash. So, with 100GiB of stored data, it's ~100 vs ~200 **MEGA**bytes overhead - and it's if no compression was used. Negligeble in practice.
> - `--features` enables `block-group-tree`, which speeds up mounting with A LOT of fragmented data... which is realistic for an FS with a ton of small files on many subvolumes, plus regular snapshots.
</details>

---

## 3. Clone temp install to the main SSD

### 3.1. Make sure everything is formatted properly

#### Manually from terminal
If something was messed up on the previous step, and we add the broken partition to fstab/crypttab - the current installation might get unable to boot. So, before we do - let's test the mounting manually:
- Make sure that the password for LUKS was typed correctly, and the partition **CAN** be unlocked - by [closing it and reopening LUKS partition](#luks-on-ssds)
- Open `fstab` with text editor: `kate /etc/fstab`
- It should contain mounting points for the **current** (intermediate) installation. Something like:
  <details>
  <summary>▶️ fstab ◀️</summary>
  
  ```fstab
  # <file system>                 <mount point>  <type>  <options>  <dump>  <pass>
  UUID=8F30-47AE                   /boot          vfat    defaults,umask=0077 0 2
  
  /dev/mapper/luks-LONG_UUID_HERE  /              btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@ 0 0
  /dev/mapper/luks-LONG_UUID_HERE  /home          btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@home 0 0
  /dev/mapper/luks-LONG_UUID_HERE  /root          btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@root 0 0
  /dev/mapper/luks-LONG_UUID_HERE  /srv           btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@srv 0 0
  /dev/mapper/luks-LONG_UUID_HERE  /var/cache     btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@cache 0 0
  /dev/mapper/luks-LONG_UUID_HERE  /var/tmp       btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@tmp 0 0
  /dev/mapper/luks-LONG_UUID_HERE  /var/log       btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@log 0 0
  
  tmpfs                            /tmp           tmpfs   defaults,noatime,mode=1777 0 0
  ```
  </details>

- `luks-<LONG_UUID_HERE>` is the auto-generated name for the **decrypted** temp-install partition that was given by CachyOS installer. No need to make it readable: after migration, the installation will be on a different partition anyway, and we'll give it a proper name.
- For now, all we need from here is to copy the long sequence of mounting options, except subvol: `defaults,noatime,ssd,compress=zstd`
  - *btw, originally, `subvol=` option could come first. For readability, you should move it to the end (as shown in the example above 👆🏻) and save `fstab`.*
- Create a couple of new dirs (to mount the roots of the tmp/main BTRFSes) and temporarily mount the main BTRFS into it. As mounting options, put `rw,<options_you_copied>,subvol=/`:
  ```bash
  DECRYPTED_NAME=main_crypt_A
  sudo mkdir /_AAA
  sudo mkdir /_btrfs_root
  sudo mount -w -t btrfs -o 'rw,defaults,noatime,ssd,compress=zstd,subvol=/' "/dev/mapper/$DECRYPTED_NAME" /_AAA
  ```

> [!NOTE]
> `subvol=/` tells to mount **the very root** of BTRFS - so, even if it has subvolumes, not them get mounted, but the actual root of the FS. Omitting this option should mount the root by default, but this default could be overridden, so better mounting it explicitly.

#### During boot
We're starting to turn the temporary installation into the permanent one. While we're still in the clean tmp install, let's check that the prepared main partition is decryptable and mountable during boot:
- Open both fstab and crypttab: `kate /etc/fstab /etc/crypttab`
- Check UUIDs for both LUKS and BTRFS of the main partition:
  ```bash
  lsblk -o NAME,PARTLABEL,LABEL,SIZE,MOUNTPOINTS,UUID,FSTYPE,FSVER,MODEL
  ```
- In `crypttab`, add the main LUKS partition - similarly to how the current (tmp) LUKS is added (`none` in the third column is an optional path to keyfile). This time, use a final name for the decrypted device - something like `Cachy_A` (later, other RAID partitions would get `Cachy_B`, `Cachy_C`, etc):
  ```crypttab
  # <name>            <device>             <password> <options>
  luks-LONG_UUID_HERE  UUID=LONG_UUID_HERE  none 
  Cachy_A              UUID=MAIN_LUKS_UUID  none 
  ```
- In `fstab`, add two new mount points: for the main/tmp partition roots (not their subvolumes), with the same options - to get something like:
  ```fstab
  ...
  /dev/mapper/luks-LONG_UUID_HERE  /var/log       btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@log 0 0
  
  /dev/mapper/luks-LONG_UUID_HERE  /_btrfs_root   btrfs   defaults,noatime,ssd,compress=zstd,subvol=/ 0 0
  /dev/mapper/Cachy_A              /_AAA          btrfs   defaults,noatime,ssd,compress=zstd,subvol=/ 0 0
  
  tmpfs                            /tmp           tmpfs   defaults,noatime,mode=1777 0 0
  ```
- Save both and reboot. Since the password between tmp and main LUKS should be the same - with `systemd-boot`, you should:
  - type it once and get **both** LUKS partitions unlocked;
  - boot into desktop.
- After logging in and running `lsblk -o NAME,PARTLABEL,LABEL,SIZE,MOUNTPOINTS,UUID,FSTYPE,FSVER,MODEL`, see that:
  - the main partition is mounted under `/_AAA`. If you visit it, it's empty.
  - the tmp partition got one extra mount point - `/_btrfs_root`. Inside it, you should see existing subvolumes as folders: `@`, `@home`, `@root`, ...

<details>
<summary>▶️ What actually happens during boot, step by step ◀️</summary>

> - We're still booting from the EFI partition on the **temporary-install** SSD.
> - There, `systemd-boot` is located. It takes control and reads its configs (we haven't touched them yet).
>   - They contain UUIDs that point to the tmp-install partition - both at LUKS and BTRFS levels.
> - `systemd-boot` initializes its own mini-linux, with its filesystem living right in RAM (from `initramfs` image)
>   - `initramfs` uses the same kernel, but contains only a bare minimum of programs/binaries required to mount the real root and continue boot.
> - After initialization, `systemd-boot` follows the config: sees that the root partition is encrypted and asks us for its password.
> - After we type it, `systemd-boot` unlocks it and - according to the same boot-config - it mounts `@` subvolume as `/`. Then, it delegates control to the actual linux (at `/`).
> - The "real" linux uses "real" `systemd` as init. It checks `crypttab` and `fstab`.
>   - One of the records in both is already done (tmp LUKS is decrypted and `@` subvolume from nested BTRFS is mounted as `/`).
>   - However, **AT HIS POINT** (so, **after** the bootloader have done its job and gave away control) `systemd` sees another record in crypttab - the one we've manually added.
>   - It attempts decrypting it with the same password, passed by `systemd-boot`. If passwords for both partitions are the same, the second (main) LUKS gets quitely decrypted, too - despite we haven't added it to the **boot** configs yet.
>   - After all partitions from `crypttab` are unlocked, they're available under `/dev/mapper/*` with the given names. Now, these virtual devices are mounted by `systemd` according to `fstab`.
> - After everything is mounted, we eventually boot into desktop.
</details>

#### Check metadata on both BTRFS partitions
For both `/_AAA` and `/_btrfs_root`
```bash
MOUNT_PATH=/_AAA
sudo btrfs filesystem show "$MOUNT_PATH"
sudo btrfs filesystem df "$MOUNT_PATH"
sudo btrfs filesystem usage "$MOUNT_PATH"
sudo btrfs subvolume list "$MOUNT_PATH"
```

<details>
<summary>▶️ What you should get for /_AAA (main BTRFS) ◀️</summary>

> - `filesystem show`:
>   - `Label: 'CachyOS'`
>   - `uuid:` the UUID of your main BTRFS
>   - `Total devices 1`
>   - Only device with `devid 1` is listed and it points to `/dev/mapper/Cachy_A`
> - `filesystem df` / `btrfs filesystem usage`:
>   - `Data, single`
>   - `System, DUP`
>   - `Metadata, DUP`
>   - The numbers of the used storage are something tiny - like, megabytes.
>   - It's normal for either of Data/System/Metadata to have `total` around 1GiB. It's not **actually** used yet - it's merely allocated. What's actually used is under `used=`, and it should be very close to 0.
</details>

<details>
<summary>▶️ ... and for /_btrfs_root (intermediate BTRFS with the installed CachyOS that we've currently booted into) ◀️</summary>

> - `filesystem show`:
>   - Whatever label was created by CachyOS installer (probably `none`)
>   - `uuid:` the UUID of the temp BTRFS
>   - Only one device, too - but points to `/dev/mapper/luks-LONG_UUID_HERE`
> - `filesystem df` / `btrfs filesystem usage`:
>   - The same data formats: `Data, single`, `System, DUP`, `Metadata, DUP`
>   - But the actual size used should be something meaningful: hundreds of MiB for metadata, multiple GiB for data.
</details>

`subvolume list` - gives empty result for `/_AAA` (no subvolumes yet), and for `/_btrfs_root` it should list all the subvolumes, something like:
```
ID 256 gen 2310 top level 5 path @
ID 257 gen 2310 top level 5 path @home
ID 258 gen 329 top level 5 path @root
ID 259 gen 28 top level 5 path @srv
ID 260 gen 2279 top level 5 path @cache
ID 261 gen 2298 top level 5 path @tmp
ID 262 gen 2310 top level 5 path @log
ID 263 gen 30 top level 256 path @/var/lib/portables
ID 264 gen 30 top level 256 path @/var/lib/machines
```

#### Flatten subvolumes

<details>
<summary>▶️ The last two entries in the example output 👆🏻 need explanation ◀️</summary>

> Note that they're not `/@var/...`, but `@/var/...` (`@/`, not `/@`) - i.e., these are subvolumes created inside another subvolume. Specifically, they are under `@` subvolume (which is mounted as `/`), inside its `var/lib` subfolder, and there - they're named as `portables` and `machines`, displayed as regular folders, but **they are subvolumes indeed**. So they live under `@/var/lib/` without any extra mounting.
> 
> In general, `@` is usually the "main" subvolume, and others are created to:
> - exclude some nested subfolders from snapshots on the parent subvolume (`@cache` has cached downloads for pacman packages and there's no point to include them into snapshot);
> - survive though parent subvolume being restored from snapshot (so, `@log` mounted as `/var/log` preserves all the logs even after `@` restore);
> - have their snapshot/restoration separate from another one (e.g., you probably want `/home` and `/root` to have their snapshots, too - but you also want them to be snapshotted/restored independently from `/`).
> 
> Whenever snapshot is created, any nested subvolumes are excluded from it... so, if `@` is snapshotted with some subvolumes created literally as subfolders inside it - if `@` is restored later, those subvolumes are lost.
> 
> This is why both these nested subvolumes should be at the top level, and just be mounted to these paths. I.e.:
> - `@/var/lib/machines` ->  `@var_machines`, mounted at `/var/lib/machines`
> - `@/var/lib/portables` -> `@var_portables`, mounted at `/var/lib/portables`
</details>

Looks like they weren't even created by CachyOS' installer and instead were added by `systemd` on first boot. Anyway, we should fix this now - before any snapshots are created.

1. Clone the existing nested subvolumes to the root level. This cloning is actually done via creation of a snapshot (every subvolume is effectively a snapshot, just a writeable one):
  ```bash
  sudo btrfs subvolume snapshot /_btrfs_root/@/var/lib/machines /_btrfs_root/@var_machines
  sudo btrfs subvolume snapshot /_btrfs_root/@/var/lib/portables /_btrfs_root/@var_portables
  ```
2. Verify that they are indeed created (`@var_machines` and `@var_portables` should appear in the end):
  ```bash
  sudo btrfs subvolume list /_btrfs_root
  ```
3. Include them to `fstab`:
  <details>
  <summary>▶️ fstab ◀️</summary>
  
  ```fstab
  ...
  /dev/mapper/luks-LONG_UUID_HERE  /var/log            btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@log 0 0
  /dev/mapper/luks-LONG_UUID_HERE  /var/lib/machines   btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@var_machines 0 0
  /dev/mapper/luks-LONG_UUID_HERE  /var/lib/portables  btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@var_portables 0 0
  
  /dev/mapper/luks-LONG_UUID_HERE  /_btrfs_root        btrfs   defaults,noatime,ssd,compress=zstd,subvol=/ 0 0
  /dev/mapper/Cachy_A              /_AAA               btrfs   defaults,noatime,ssd,compress=zstd,subvol=/ 0 0
  
  tmpfs                            /tmp                tmpfs   defaults,noatime,mode=1777 0 0
  ```
  </details>

4. Reboot and verify that they're shown as mounted:
  ```bash
  lsblk -o NAME,PARTLABEL,LABEL,SIZE,MOUNTPOINTS,UUID,FSTYPE,FSVER,MODEL
  ```
5. Check that they truly are mounted:
  <details>
  <summary>▶️ Terminal ◀️</summary>
  
  ```bash
  # both are likely empty for now:
  sudo ls -hlA /var/lib/machines
  sudo ls -hlA /var/lib/portables
  
  # create a dummy file in each - under actual subvolume, not mounting point:
  sudo touch /_btrfs_root/@var_machines/qqq_zzz_aaa
  sudo touch /_btrfs_root/@var_portables/www_xxx_sss
  
  # now, under mount point, both should show the same files:
  sudo ls -hlA /var/lib/machines
  sudo ls -hlA /var/lib/portables
  
  # clean up:
  sudo rm /var/lib/machines/qqq_zzz_aaa
  sudo rm /var/lib/portables/www_xxx_sss
  ```
  </details>

6. Remove old nested subvolumes:
  <details>
  <summary>▶️ Terminal ◀️</summary>
  
  ```bash
  # to check, list all subvolumes before the operation:
  sudo btrfs subvolume list /_btrfs_root
  
  # delete:
  sudo btrfs subvolume delete -c /_btrfs_root/@/var/lib/machines
  sudo btrfs subvolume delete -c /_btrfs_root/@/var/lib/portables
  
  # list afterwards - there should be no nested subvolumes now:
  sudo btrfs subvolume list /_btrfs_root
  ```
  </details>

7. Reboot once again - to double-check that nothing is broken.

If everything is fine, we're ready to **actually** migrate.

### 3.2. Clone subvolumes 

> [!NOTE]
> For all other snippets in this step:
> ```bash
> CLONED_SUBVOLS=(@ @home @root @srv @cache @tmp @log @var_machines @var_portables)
> SRC_BTRFS=/_btrfs_root
> NEW_BTRFS=/_AAA
> ```

#### Make read-only snapshots
First, we need to "freeze" the current state of the running intermediate system. Same creation of snapshots - but this time, **actual** snapshots (i.e., read-only, with `-r` argument).

```bash
for sv in "${CLONED_SUBVOLS[@]}"; do
  sudo btrfs subvolume snapshot -r "$SRC_BTRFS/$sv" "$SRC_BTRFS/__$sv"
done
```

Verify that the snapshots starting with `__` are created (`__@`, `__@home`, ...):
```bash
sudo btrfs subvolume list "$SRC_BTRFS"
```

... and that they truly are read-only:
```bash
for sv in "${CLONED_SUBVOLS[@]}"; do
  echo "\n$sv:"
  sudo btrfs subvolume show "$SRC_BTRFS/$sv" | grep -i flags
  echo "__$sv:"
  sudo btrfs subvolume show "$SRC_BTRFS/__$sv" | grep -i flags
done
```

#### Clone read-only snapshots to main BTRFS
Now that they're read-only, we can utilize BTRFS' send-receive feature which allows truly clone entire subvolumes from one BTRFS to another:
```bash
for sv in "${CLONED_SUBVOLS[@]}"; do
  echo "\nSending __$sv..."
  sudo btrfs send "$SRC_BTRFS/__$sv" | sudo btrfs receive "$NEW_BTRFS"
done
```
> [!WARNING]
> The process takes time - don't interrupt it, wait for the completion.
> 
> To see the progress of **every file** being sent - add `-v` argument after `receive`.

Then verify that the read-only snapshots are created on the main BTRFS:
```bash
sudo btrfs subvolume list "$NEW_BTRFS"
```

#### Turn read-only snapshots on main BTRFS back to normal subvolumes
```bash
for sv in "${CLONED_SUBVOLS[@]}"; do
  sudo btrfs subvolume snapshot "$NEW_BTRFS/__$sv" "$NEW_BTRFS/$sv"
done
```

#### Remove temporary read-only snapshots on main BTRFS
```bash
for sv in "${CLONED_SUBVOLS[@]}"; do
  sudo btrfs subvolume delete -c "$NEW_BTRFS/__$sv"
done
```

... and check that only the proper subvolumes are there now:
```bash
sudo btrfs subvolume list /_AAA
```

We've transferred the entire linux installation to the main encrypted BTRFS! Now, we need to update the disk UUIDs to make it bootable.

### 3.3. Update UUIDs on main BTRFS

For clarity: we are still running Linux from the **intermediate** BTRFS, but - from within it - we're now configuring the migrated installation on the **main** BTRFS.

To check what are we currently booted into, use `lsblk`:
```bash
clear && lsblk -o NAME,PARTLABEL,LABEL,SIZE,MOUNTPOINTS,UUID,FSTYPE,FSVER,MODEL
```

#### `fstab`
Open it **on the main BTRFS**:
```bash
kate /_AAA/@/etc/fstab
```

Replace every occurrence of `luks-<LONG_UUID_HERE>` with `Cachy_A`. And, since the root of `Cachy_A` will take `/_btrfs_root`, remove the `/_AAA` mount point:

<details>
<summary>▶️ fstab ◀️</summary>

```fstab
# <file system>                 <mount point>       <type>  <options>  <dump>  <pass>
UUID=8F30-47AE       /boot               vfat    defaults,umask=0077 0 2

/dev/mapper/Cachy_A  /                   btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@ 0 0
/dev/mapper/Cachy_A  /home               btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@home 0 0
/dev/mapper/Cachy_A  /root               btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@root 0 0
/dev/mapper/Cachy_A  /srv                btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@srv 0 0
/dev/mapper/Cachy_A  /var/cache          btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@cache 0 0
/dev/mapper/Cachy_A  /var/tmp            btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@tmp 0 0
/dev/mapper/Cachy_A  /var/log            btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@log 0 0
/dev/mapper/Cachy_A  /var/lib/machines   btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@var_machines 0 0
/dev/mapper/Cachy_A  /var/lib/portables  btrfs   defaults,noatime,ssd,compress=zstd,subvol=/@var_portables 0 0

/dev/mapper/Cachy_A  /_btrfs_root        btrfs   defaults,noatime,ssd,compress=zstd,subvol=/ 0 0

tmpfs                /tmp                tmpfs   defaults,noatime,mode=1777 0 0
```
</details>

> [!NOTE]
> The `/boot` still points to **the same** EFI partition - i.e., the one that lives on the intermediate SSD.

#### `crypttab`
```bash
kate /_AAA/@/etc/crypttab
```

Remove the `luks-LONG_UUID_HERE` entry, leaving only the `Cachy_A` cryptdevice. Additionally, it's worth adding `luks2,discard` decryption options to the 4th column - to explicitly decrypt it as LUKS2 and allow discards (TRIM) for the SSD:
```crypttab
# <name> <device>             <password> <options>
Cachy_A   UUID=MAIN_LUKS_UUID  none       luks2,discard
```

#### Intermediate `systemd-boot` configs
https://wiki.archlinux.org/title/Systemd-boot#Adding_loaders

For now, we're gonna use the same bootloader from the temp-EFI - to boot into the main-BTRFS-Linux. For that, we're gonna create the respective boot entries:
1. With file explorer, go to: `/boot/loader/entries/`. Dolphin should suggest to switch to Administrator to see any contents of `/boot`.
  - If not - put into the path field: `admin:///boot/loader/entries/`
2. Two files should be there - likely, `linux-cachyos.conf` and `linux-cachyos-lts.conf`.
3. Copy them with some suffix (e.g., `-NEW`): `linux-cachyos-NEW.conf` and `linux-cachyos-NEW-lts.conf`.
4. Open them with Kate.

Now, we need to make the same modifications in both files:
- The first line should be `title ...` - add the same unique suffix there, e.g. `(NEW)`
- The main line that tells WHERE to boot into is the `options ...` line:
  - use `lsblk` command above to identify the UUIDs
  - `root=UUID=<tmp-BTRFS UUID>` - replace it with `root=UUID=<main-BTRFS UUID>` or just `root=/dev/mapper/Cachy_A`
  - verify that `rootflags=` has the same options as in the fstab for `/`: `rootflags=defaults,noatime,ssd,compress=zstd,subvol=/@`
  - `rd.luks.name=<tmp-LUKS UUID>=luks-SAME_LONG_UUID` - this tells what LUKS partition must be decrypted and what name under `/dev/mapper` it will get. Replace it with `rd.luks.name=<main-LUKS UUID>=Cachy_A`

Example result:
```
title Linux CachyOS (NEW)
options root=/dev/mapper/Cachy_A rw rootflags=defaults,noatime,ssd,compress=zstd,subvol=/@ rd.luks.name=<your main-LUKS UUID>=Cachy_A zswap.enabled=0 nowatchdog quiet splash
```

#### Verify boot
Reboot, choosing the new option in the bootloader. The same LUKS password should work as before. After login - check with `lsblk` that all the linux dirs (`/`, `/home`, `/root`, ...) are now mounted on the **MAIN** BTRFS.

### 3.4. Migrate bootloader

> [!NOTE]
> Now, the actual Linux installation is fully migrated. But the bootloader (`systemd-boot`) still lives on the temp SSD's EFI partition.

#### Change `/boot` mount point
While booted into the main installation:
- Go to `/boot` in file explorer and check that it's populated with files/folders from the tmp-SSD-EFI.
- Open `fstab`:
```bash
kate /etc/fstab
```
- Use `lsblk` to know UUID of EFI partition on the main SSD.
- In `fstab`:
  - Duplicate the `/boot` line.
  - On one of them, keep it mounted as `/boot` but replace the old UUID with the main-EFI UUID.
  - On the other, keep the old UUID, but mount it as (now free again) `/_AAA`.
- Save and reboot (again, to the **main/new** SSD).
- After reboot, verify with `lsblk` that both EFI partitions have the expected mount points (new-main as `/boot`, old-tmp as `/_AAA`). Then open both in file explorer to confirm their contents (`/boot` is empty now or contains just a file or two, `/_AAA` shows the old bootloader files).
- If anything is broken/wrong, the intermediate installation on temp SSD (both EFI and LUKS+BTRFS) is still there to restart from earlier steps.

> [!WARNING]
> We're in a very unusual state right now:
> - the **running** Linux is on main-SSD,
> - but it's **booted** from temp-SSD,
> - and the **mounted** `/boot` isn't even pointing at it - instead, it points to (empty yet) main-SSD's EFI.

To ensure that:
```bash
# This command should:
# - show the option we're booted through with `(selected)` suffix;
# - show all the boot options for it;
# - also highlight it with a red `(reported/absent)` suffix - i.e., such config not fount under the current /boot
sudo bootctl list
# Arrows/PgUp/PgDn to navigate
# `q` button to exit

# This one should give even more detailed log about boot process:
sudo bootctl status
```

#### Copy system files
Some of the files on EFI partition are generated for your individual installation, but some come with system packages, installed with `pacman` (e.g., `linux` kernel itself or microcode for AMD/intel).

> Copy all the **files** (not folders) on the root EFI - from `/_AAA` to `/boot`.

Likely, there will be:
- `initramfs-linux-cachyos.img` / `initramfs-linux-cachyos-lts.img` (could skip, we'll regenerate them next)
- `amd-ucode.img` / `intel-ucode.img`
- `vmlinuz-linux-cachyos` / `vmlinuz-linux-cachyos-lts`

#### Regenerate initramfs
```bash
sudo mkinitcpio -P
```

#### Install bootloader to the main SSD
https://wiki.archlinux.org/title/Systemd-boot#Installing_the_UEFI_boot_manager

`--efi-boot-option-description` allows to specify how this EFI bootloader will be named in UEFI boot menu.

```bash
sudo bootctl install --esp-path=/boot --efi-boot-option-description="CachyOS bootloader (systemd-boot)"
```

It should create two subfolders with files under `/boot`: `EFI` and `loader`.

#### Copy bootloader configs
From `/_AAA` to `/boot`, copy (overwrite):
- `loader/entries.srel`
- `loader/loader.conf`
- Our duplicated/"new" entry-configs from `loader/entries/`
  - Rename them, removing "new" - both from filename and from within the config.
  - Alternatively, `sudo sdboot-manage --esp-path=/boot gen` could be used to generate the default boot entries - according to `/etc/sdboot-manage.conf`.

#### Change UEFI boot order
- Reboot to the motherboard's UEFI interface (`DEL` on POST screen).
- Change boot order, so the just-installed bootloader would be first. `systemd-boot` installs two of them, so in our case they should be shown as:
  - `CachyOS bootloader (systemd-boot)`
  - `CachyOS bootloader (systemd-boot) Fallback`
- Save&Reboot (`F10`)
- Boot into the main installation - now also via EFI partition on the same main SSD.
- From inside the booted OS, double-check that this is indeed how you've booted - by calling `sudo bootctl list` and/or `sudo bootctl status` again.
  - The red `(reported/absent)` status-indicators should now disappear from the `sudo bootctl list` output.

---

## 4. Clean up the intermediate SSD

### 4.1. Unlink the main installation from it

- ⚠️ Remove `/_AAA` line from `fstab` and reboot ⚠️
- Double-check with `lsblk` that no partition from intermediate SSD is mounted or decrypted, and `/_AAA` dir isn't shown anywhere in the output. To make absolutely sure, also:
  ```bash
  # Should give an empty output - nothing is mounted under this path:
  sudo mount | grep -i '_AAA'
  ```
- Remove the `/_AAA` dir itself.
  ```bash
  sudo rm -rf '/_AAA'
  ```
- Reboot once again to make sure the main installation still boots.

### 4.2. Actually clear the tmp SSD

Following the same steps as in [0.2. SSD secure erase](#02-ssd-secure-erase), do the full secure erase for it.

---

## 5. Expand BTRFS to additional SSDs for RAID

### 5.1. Partition + set up LUKS on extra SSDs

According to 2.1 - 2.2:
- [Partition them](#21-partitioning) (EFI + linux partition - same sizes)
- [Do `luksFormat`](#22-luks-on-main-partition), same options, same password.
- [Open all encrypted partitions](#luks-on-ssds) to double-check the password is correct + store the `allow-discards` flag.
  - Verify it's set on each with `luksDump`.
- If everyfing as expected, update `/etc/crypttab`:
  - Add the extra decrypted devices (`Cachy_B`, `Cachy_C`, ...);
  - Set their UUIDs from the respective partitions (use `lsblk`).
- Reboot & check with `lsblk` that the devices are indeed decrypted under these names.

### 5.2. Update boot configs with new devices

Currently, the configs under `/boot/loader/entries/*` have only one `rd.luks.name=` option, for `Cachy_A`. This seems fine for now, because after full boot to desktop we see all LUKS partitions decrypted, but it's actually sketchy.

For now, BTRFS still uses only one device. So, on boot, the bootloader (`systemd-boot`) decrypts only it, then mounts it as `/`, then passes control to `systemd` which finalizes boot/init process - including the decryption of other LUKS devices, which are then visible as we boot to desktop. However...

> [!CAUTION]
> If we leave configs like this, then **AFTER** main BTRFS gets expanded to multiple SSDs, it will be unable to boot: only one LUKS partition will be decrypted at `systemd-boot` stage. And for successful mounting of BTRFS **all** the underlying block devices must be visible. So `systemd-boot` would fail to mount `/` and thus won't be able to continue the boot process.

This is why, even though everything works seemingly fine **now**, we need to add `rd.luks.name=` options for **all** the LUKS partitions that the root BTRFS gonna take later.
- Open `/boot/loader/entries` in file manager.
- Open all the config files stored thare in text editor (Kate).
- All the boot options are listed as one line, and in the middle of it there is `rd.luks.name=<UUID>=Cachy_A` argument.
- After it, add similar arguments for all other `Cachy_*` LUKS partitions, separated with spaces.
- To avoid typos, copy the same cryptdevice names and UUIDs from `/etc/crypttab`.
- After updating and saving all config files, reboot once again to check they still work.

### 5.3. Verify that BTRFS has only one device yet

```bash
sudo btrfs filesystem show /_btrfs_root
sudo btrfs filesystem usage /_btrfs_root
```

For `show`, there will be something like:
```
Label: 'CachyOS'  uuid: <your main-BTRFS UUID>
        Total devices 1 FS bytes used 8.48GiB
        devid    1 size 99.98GiB used 12.02GiB path /dev/mapper/Cachy_A
```

`usage` will be more detailed, but in the end, under `Data,single:`, `Metadata,DUP:`, `System,DUP:`:
- Note those single/dup types.
- Under each - again, only the main crypt-device should be listed.

### 5.4. Add extra SSDs to main BTRFS

Replace `Cachy_B Cachy_C` with your actual list of crypt-devices:
```bash
LUKS_DEVICES=(Cachy_B Cachy_C)
for cryptDev in "${LUKS_DEVICES[@]}"; do
  echo "\nAdding: $cryptDev"
  sudo btrfs device add "/dev/mapper/$cryptDev" /_btrfs_root
done
```

After it completes, check again with the same commands that the devices are indeed added (but they don't participate in the actual data redundancy yet):
```bash
sudo btrfs filesystem show /_btrfs_root
sudo btrfs filesystem usage /_btrfs_root
```

```
Label: 'CachyOS'  uuid: <your main-BTRFS UUID>
        Total devices 3 FS bytes used 8.48GiB
        devid    1 size 99.98GiB used 12.02GiB path /dev/mapper/Cachy_A
        devid    2 size 99.98GiB used 0.00B path /dev/mapper/Cachy_B
        devid    3 size 99.98GiB used 0.00B path /dev/mapper/Cachy_C
```

Also, check `lsblk` - it should show the same UUID/FS-label for all the devices that BTRFS spans over.

### 5.5. Convert BTRFS to RAID1

https://btrfs.readthedocs.io/en/latest/mkfs.btrfs.html#man-mkfs-profiles

Currently, devices **are** added, but they don't act as a RAID-array. To do it, we need to rebalance the FS to a different storage format. Depending on how many SSDs are used, it would be different:
- 2: `raid1`
- 3: `raid1c3`
- 4+: `raid1c4`

> [!WARNING]
> This is somewhat-dangerous operation. You should make precautions to avoid power outage during rebalance.
> 
> If power goes out in the middle, the FS stays in a semi-broken state. Normally, it's not catastrophic and could be recovered, but you **DEFINITELY** should avoid it with all you can.

Replace `raid1c3` with the raid type for your number of SSDs:
  - `-mconvert` sets the type for metadata (and consequently, for sysem data) - it should be as high-level RAID as you can have.
  - `-dconvert` does the same for the actual stored data. You could give it a lower-level raid to get more of usable space at the cost of less redundancy... but unless you're VERY tight in space, you shouldn't do it - it kinda kills the whole point.

```bash
sudo btrfs balance start -dconvert=raid1c3 -mconvert=raid1c3 --full-balance --enqueue -v /_btrfs_root
```

While it does its job - to see the progress (it's reversed: goes from `100% left` to complete) - open a second terminal:
```bash
watch -n 0.5 sudo btrfs balance status /_btrfs_root
```

When it completes, check the BTRFS status again - now it should show both all your devices attached **and** them using the RAID mode:
```bash
sudo btrfs filesystem usage /_btrfs_root
```

Reboot once again to verify that system still works.

### 5.6. Clone EFI partitions

#### Add mount folders
```bash
sudo mkdir /.boot_A
sudo mkdir /.boot_B
sudo mkdir /.boot_C

# This one won't be mounted into, and will store a copy inside the main BTRFS:
sudo mkdir /_boot_bak
```

#### Add them to `fstab`
In `fstab`, duplicate the first line as many times as many RAID-SSDs you have. Edit their UUIDs to point to the respective EFI partitions. Yes, the main one will be mounted twice: under actual `/boot` and under `/.boot_A`, for consistency.

Reboot to check it boots + verify with `lsblk` that they're mounted

#### Backup the working EFI partition
Copy the entire contents of `/boot` to all the created/mounted folders.

If the boot itself breaks later, you'll have a working EFI partition on another SSD. And if all of them fail (your friend/roommate/spouse/kids make a "joke" of wiping all boot partitions), one extra copy will be saved inside the main BTRFS partition itself (which should be regularly backed up to other device), and would be easy to restore.

It might be automated to sync `/boot` with all these copies (e.g., a custom systemd service with a simple `rsync` command running with some long-enough delay after boot - only when the boot was successful). For now, just copy them manually from time to time.

---

Finally, done.
