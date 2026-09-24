# Snapper (on Arch/CachyOS)

> [!TIP]
> It's helpful to install `btrfs-assistant` - allows doing some stuff with GUI.

## 1. `snapper` itself

1. Install `snapper`.
2. Create configs for each of the subvolumes that contain the actual OS (not data/downloads/logs).
    ```bash
    sudo snapper -c 'root' create-config /
    sudo snapper -c 'home' create-config /home
    sudo snapper -c 'root_user' create-config /root
    ```
    
    - The actual config files are stored at: `/etc/snapper/configs/`
    - They're registered for Snapper in: `/etc/conf.d/snapper`
3. Edit them:
    - `kate /etc/snapper/configs/*`
    - Specifying how many snapshots to keep (`NUMBER_*`, `TIMELINE_*`).
    - Allow admins to run it (`ALLOW_USERS`, `ALLOW_GROUPS`).
    
    E.g.:
    
    <details>
    <summary>▶️ /etc/snapper/configs/root ◀️</summary>
    
    ```bash
    # subvolume to snapshot
    SUBVOLUME="/"
    
    # filesystem type
    FSTYPE="btrfs"
    
    # btrfs qgroup for space aware cleanup algorithms
    QGROUP=""
    
    # fraction or absolute size of the filesystems space the snapshots may use
    SPACE_LIMIT="0.5"
    
    # fraction or absolute size of the filesystems space that should be free
    FREE_LIMIT="0.2"
    
    # users and groups allowed to work with config
    ALLOW_USERS="drl"
    ALLOW_GROUPS="wheel"
    
    # sync users and groups from ALLOW_USERS and ALLOW_GROUPS to .snapshots
    # directory
    SYNC_ACL="no"
    
    # start comparing pre- and post-snapshot in background after creating
    # post-snapshot
    BACKGROUND_COMPARISON="yes"
    
    # run daily number cleanup
    NUMBER_CLEANUP="yes"
    
    # limit for number cleanup
    NUMBER_MIN_AGE="3600"
    NUMBER_LIMIT="15"
    NUMBER_LIMIT_IMPORTANT="10"
    
    # create hourly snapshots
    TIMELINE_CREATE="yes"
    
    # cleanup hourly snapshots after some time
    TIMELINE_CLEANUP="yes"
    
    # limits for timeline cleanup
    TIMELINE_MIN_AGE="3600"
    # DRL: add some overlap from smaller to bigger time frames
    TIMELINE_LIMIT_HOURLY="18"
    TIMELINE_LIMIT_DAILY="10"
    TIMELINE_LIMIT_WEEKLY="5"
    TIMELINE_LIMIT_MONTHLY="5"
    TIMELINE_LIMIT_QUARTERLY="5"
    TIMELINE_LIMIT_YEARLY="5"
    
    # cleanup empty pre-post-pairs
    EMPTY_PRE_POST_CLEANUP="yes"
    
    # limits for empty pre-post-pair cleanup
    EMPTY_PRE_POST_MIN_AGE="3600"
    ```
    </details>
    
4. Enable systemd service:
    
    ```bash
    # Show all units related to snapper:
    sudo systemctl list-unit-files | grep -i snapper
    
    # Enable the main snapper timer (runs every hour):
    sudo systemctl enable --now snapper-timeline.timer
    ```

## 2. `snap-pac`

1. Install `snap-pac`.
    - Actual files it installs: `sudo pacman -Ql snap-pac`
    - Its hooks are under `/usr/share/libalpm/hooks/`
2. Add all snapper configs to the list of what `snap-pac` snapshots on pacman operations:
    - `kate /etc/snap-pac.ini`
    - Create a section for each config, with `snapshot = True` in it.
    - In `[DEFAULT]` section, `important_packages` and `important_commands` could pe specified to mark snapshots as important.
    
    E.g.:
    ```ini
    [DEFAULT]
    important_packages = ["mkinitcpio", "linux", "linux-lts", "linux-api-headers", "linux-cachyos", "linux-cachyos-headers", "linux-cachyos-lts", "linux-cachyos-lts-headers"]
    important_commands = ["pacman -Syu", "pacman -Syyu", "pacman -Syuu", "pacman -Syyuu"]
    
    [root]
    snapshot = True
    
    [root_user]
    snapshot = True
    
    [home]
    snapshot = True
    ```

## 3. Including `/boot` partition with `rsync`

Bootloader is located on EFI partition - it's mounted as:
- `/boot` (`systemd-boot` style): the whole boot, including kernels, initramfs, etc.
- `/boot/efi` (`GRUB` style): only the actual bootloader on it, while `/boot` is on the main partition.

Either way, a crucial part of the system isn't included to the snapshots. It's worth having it backed up, too.

### 3.1. `pacman` hooks for backup on system updates

Similarly to how `snap-pac` works, let's make our own hooks to clone the current state of `/boot` to a folder **inside** the main BTRFS partition. Here, it's `/_boot-bak`.

1. Install `rsync`.
2. Create the backup directory: `sudo mkdir /_boot-bak`.
3. Manually clone the whole `/boot` there, to make sure it works: `sudo rsync -acv --delete /boot/ /_boot-bak`. Args:
    - `-a`: preserves all the attributes (access rights, ownership, mod-time).
    - `-c`: compares source/target files by their actual contents (via checksums) instead of just mod-time. Currently, doesn't matter, but in the future ensures that only the files with **actually changed contents** are overwritten in the backup - even if their dates mismatch.
    - `-v`: print what's actually copied.
    - `--delete`: also deletes the files that no longer exist in the source (so actually makes the backup identical, with no outdated leftovers).
    - `/boot/`: source to back up from. The `/` at the end is essential to map the source directory to the backup one. Without it, `/boot` will be cloned into `/_boot-bak/boot`.
    - `/_boot-bak`: target to to back up to. Doesn't matter if it has the trailing slash or not.
4. Create files for pacman hooks:
    - Under `/etc/pacman.d/hooks/`.
    - One should run before the `snap-pac`'s pre-snapshot (backs up the current `/boot` as it was before the update). So it should be named accordingly (`00-*`) and have `When = PreTransaction` under `[Action]`.
    - Another one runs after the installation is complete, but just before the `snap-pac`'s post-snapshot (to reflect the updated state). `zy-*` is a good prefix + `When = PostTransaction`.
    - Both should error out if the source isn't a mount point but is just a regular dir. So let's make the actual exec command as: `mountpoint -q /boot && rsync ...`
    - It's better to trigger them whenever any package gets installed/updated, since it's EXTREMELY hard to list only those that end up modifying contents of `/boot`. So `Target = *`, like in `snap-pac`.
    - Under `[Action]`, both can have `Depends = rsync` to only run them when `rsync` is present. With that line, the hooks would be just silently skipped if rsync isn't installed. Without, they'll explicitly error out (my choice).
  
    ```ini
    # /etc/pacman.d/hooks/00-DRL-boot-backup-pre.hook
    
    [Trigger]
    Operation = Install
    Operation = Upgrade
    Operation = Remove
    Type = Package
    Target = *
    
    [Action]
    Description = Syncing /boot to /_boot-bak (before pacman transaction)...
    # Depends = rsync
    When = PreTransaction
    Exec = /usr/bin/sh -c '/usr/bin/mountpoint -q /boot && /usr/bin/rsync -acv --delete /boot/ /_boot-bak'
    ```
  
    ```ini
    # /etc/pacman.d/hooks/zy-DRL-boot-backup-post.hook
    
    [Trigger]
    Operation = Install
    Operation = Upgrade
    Operation = Remove
    Type = Package
    Target = *
    
    [Action]
    Description = Syncing /boot to /_boot-bak (just before post-snapshot)...
    # Depends = rsync
    When = PostTransaction
    Exec = /usr/bin/sh -c '/usr/bin/mountpoint -q /boot && /usr/bin/rsync -acv --delete /boot/ /_boot-bak'
    ```

5. Run an update (if available) or reinstall some tiny package to make sure that the whole stack works (`snapper` + `snap-pac` + our boot-backup hooks).

### 3.2. `systemd` units for stable backups

A `/boot` which is auto-backed-up into snapshots is good. An **always** stable boot is better.

Let's create a systemd timer which **also** backs up `/boot` only **after** we've successfully booted into the actual OS. To be safe, let's do it after a good delay: 30min-1h of running Linux.

If the PC has multiple physical drives (SSDs/HDDs), the ideal solution would be:
- the main `/boot` **is** the main EFI (`systemd-boot` style - works even for GRUB);
- each other drive in the PC has its own EFI partition, too;
- we back up `/boot` that we've **successfully** booted from into them.

This way, if an update messes up the main `/boot`, the system can still be booted from a backup EFI on another SSD/HDD. Alternatively, the stable `/boot` can just be backed up to a regular directory inside the main BTRFS. But in a case of broken `/boot`, a live disk would be required for recovery.

Create two systemd unit files:

#### Service

```ini
# /etc/systemd/system/DRL-boot-stable-backup@.service

[Unit]
Description=Sync successfully-booted /boot to %f
ConditionPathIsMountPoint=%f
RequiresMountsFor=%f

[Service]
Type=oneshot
Nice=19
IOSchedulingClass=idle
#KillSignal=SIGINT
# Refuse to run (loudly) if /boot isn't mounted, the destination doesn't exist yet, or it's "/"
ExecStartPre=/usr/bin/mountpoint -q /boot
ExecStartPre=/usr/bin/test -d %f
ExecStartPre=/usr/bin/test %f != /
ExecStartPre=/usr/bin/mountpoint -q %f
ExecStart=/usr/bin/rsync -acv --delete /boot/ %f
```

The service is designed for a case with backup being an alternative EFI partition on other drive. For a regular folder, remove/comment these lines:

```ini
...

ConditionPathIsMountPoint=%f
RequiresMountsFor=%f

...

ExecStartPre=/usr/bin/mountpoint -q %f
```

#### Timer

```ini
# /etc/systemd/system/DRL-boot-stable-backup@.timer

# Enable with:
# sudo systemctl enable --now "$(systemd-escape --template=DRL-boot-stable-backup@.timer --path /path/to/mounted/backup/boot/partition)"

[Unit]
Description=Sync stable /boot to %f, 30-60 min after boot

[Timer]
OnBootSec=30min
RandomizedDelaySec=30min

[Install]
WantedBy=timers.target
```

This timer runs at some random point between 30mins and 1h since the system is booted.

#### Enable backup

1. Create the mount point for the EFI clone. E.g.: `sudo mkdir '/.boot_B'`
2. Add the secondary EFI to `fstab`, by:
    - duplicating the `/boot` line (main EFI),
    - replacing it with UUID of secondary EFI,
    - changing mount path to `/.boot_B`
3. Reboot + use `lsblk -f` to make sure that indeed it's mounted.
4. Enable the timer with:
    ```bash
    TIMER_INSTANCE="$(systemd-escape --template=DRL-boot-stable-backup@.timer --path '/.boot_B')"
    sudo systemctl enable --now "$TIMER_INSTANCE"
    
    # Check the status:
    sudo systemctl status "$TIMER_INSTANCE"
    ```
  
    In the status, it should say:
    - `Active: active`, with a countdown (in the end) of how much time left till the actual run;
    - The first line should be: "... - Sync stable /boot to **/.boot_B**, ..." (the right path to the mounted secondary EFI)
5. Reboot and wait till the actual run:
    ```bash
    # Timer status - shows the countdown till run:
    sudo systemctl status "$(systemd-escape --template=DRL-boot-stable-backup@.timer --path '/.boot_B')"
    
    # Service status - should be empty before the timer triggers, and display the rsync's output (actual synced files) after the run:
    sudo systemctl status "$(systemd-escape --template=DRL-boot-stable-backup@.service --path '/.boot_B')"
    ```
