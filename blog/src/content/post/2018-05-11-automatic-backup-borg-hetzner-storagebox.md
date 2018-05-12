---
title: Automatic Backup with Borg to Hetzner Storagebox
#author: Jake
date: '2018-05-11'
categories:
  - Laptop
  - Server
tags:
  - Arch
  - Backup
slug: automatic-backup-borg-hetzner-storagebox
---

# Automatic Backup with Borg to Hetzner Storagebox

## Storagebox

To start some initial config has to be done. Take a look a their [wiki](https://wiki.hetzner.de/index.php/BorgBackup/en#Workflow_with_Borg) as it's describing it pretty good. 
But in short: transfer ssh key and setup a Borg repository (read ahead first). This is necessary anyway but also tests the ssh key setup.

**Note:** I had to fiddle around til my setup worked like it does now. Part of that was to switch from [user-systemd](https://wiki.archlinux.org/index.php/Systemd/User) to root-systemd execution of the backup. This included generating and uploading a root ssh key to the Storagebox.

## Borg

Before running `borg init` one should take a good look at the [Borg doc](https://borgbackup.readthedocs.io/en/stable/usage/general.html). So `borg init` has pretty sane default configs when executed with `--encryption=repokey`. 

The [example script](https://borgbackup.readthedocs.io/en/stable/quickstart.html#automating-backups) show that  the next step would be the actual backing up. `borg create` is used for that when pointed to the previously initialized repo. Refer to the link for a config example. Keep in mind [not everything on a system is supposed to be backed up](https://borgbackup.readthedocs.io/en/stable/faq.html#can-i-backup-my-root-partition-with-borg).
Next step in the script is `borg prune` so you always keep or delete backup-states based on a reasonable rule.

As always I combined several sources into my own Frankenstein-like result where I really don't know if every option is the best choice. Especially I'm not sure how to safe exporting of the passphrase is but as the systemd service is executed on root-level and the file is only on an encrypted drive I figured it's okay in my case.
Anyway here is the script:

```
#!/bin/sh

# Setup steps:
# First config ssh keys for storage box as described: https://wiki.hetzner.de/index.php/BorgBackup
# Next export BORG_RSH like below 
# And 'borg init --encryption=repokey ssh://USER@USER.your-storagebox.de:23/./backups/t480s'
# ----------------------------------------------------------------------------

# Script for automatic backup:

# Export specific ssh key to be used:
export BORG_RSH='ssh -i /root/.ssh/storagebox-from-root'

# Setting this, so the repo does not need to be given on the commandline:
export BORG_REPO=ssh://USER@USER.your-storagebox.de:23/./backups/t480s

# Setting this, so you won't be asked for your repository passphrase:
export BORG_PASSPHRASE='SUPERSECRETPASSSSSSSSSSPHRASE'

# some helpers and error handling:
info() { printf "\n%s %s\n\n" "$( date )" "$*" >&2; }
trap 'echo $( date ) Backup interrupted >&2; exit 2' INT TERM

info "Starting main backup"

# Backup the most important directories into an archive named after
# the machine this script is currently running on:
# (using lzma because hetzner storage box runs last major version without zstd)

borg create                         \
    --verbose                       \
    --filter AME                    \
    --list                          \
    --stats                         \
    --show-rc                       \
    --compression lzma,5            \
    --exclude-caches                \
    --exclude '/home/*/.cache/*'    \
    --exclude '/var/cache/*'        \
    --exclude '/var/tmp/*'          \
                                    \
    ::'{hostname}-{now}'            \
    /bin			    \
    /boot			    \
    /etc                            \
    /home                           \
    /lib			    \
    /lib64			    \
    /root                           \
    /sbin			    \
    /usr			    \
    /var                            \

# Route the normal process logging to journalctl
2>&1

backup_exit=$?

info "Pruning main repository"

# Use the `prune` subcommand to maintain 7 daily, 4 weekly and 6 monthly
# archives of THIS machine. The '{hostname}-' prefix is very important to
# limit prune's operation to this machine's archives and not apply to
# other machines' archives also:

borg prune                          \
    --list                          \
    --prefix '{hostname}-'          \
    --show-rc                       \
    --keep-daily    7               \
    --keep-weekly   4               \
    --keep-monthly  6               \

prune_exit=$?

#-----------------------------------------------------------------------------
# Almost same create+prune but for veracrypt partition as it is not always available

info "Starting veracrypt backup"

borg create                         \
    --verbose                       \
    --filter AME                    \
    --list                          \
    --stats                         \
    --show-rc                       \
    --compression lz4               \
    --exclude-caches                \
    --exclude '/home/*/.cache/*'    \
    --exclude '/var/cache/*'        \
    --exclude '/var/tmp/*'          \
                                    \
    ::'veracrypt-{now}'             \
    /mnt/veracrypt1                 \

# Route the normal process logging to journalctl
2>&1

backup_v_exit=$?

info "Pruning veracrypt repository"

borg prune                          \
    --list                          \
    --prefix 'veracrypt-'          \
    --show-rc                       \
    --keep-daily    7               \
    --keep-weekly   4               \
    --keep-monthly  6               \

prune_v_exit=$?

#-----------------------------------------------------------------------------

# Include the remaining device capacity in the log - TODO should use 
# echo "df -hl" | stfp USER@USER.your-storagebox.de 

# list repo content for log
borg list $BORG_REPO

# Unset pass
export BORG_PASSPHRASE=''

# use highest exit code as global exit code
max_backup=$(( backup_exit > backup_v_exit ? backup_exit : backup_v_exit ))
max_prune=$(( prune_exit > prune_v_exit ? prune_exit : prune_v_exit ))
global_exit=$(( max_backup > max_prune ? max_backup : max_prune ))

if [ ${global_exit} -eq 1 ];
then
    info "Backup and/or Prune finished with a warning"
fi

if [ ${global_exit} -gt 1 ];
then
    info "Backup and/or Prune finished with an error"
fi

exit ${global_exit}

```

Note: I have one additional partition I want to backup separately so the script has repetitive parts.

When this setup is working one of two requirements are met. The backup is done with a modern and well knows software including encryption and other perks. But what is a backup if only run manually and (in my experience) therefore not often enough? Right, useless. So let's automate things.

## Service file

As I'm using Arch Linux the way to go to automate process is systemd. Luckily I found a [blog post](https://blog.andrewkeech.com/posts/170719_borg.html) addressing automation of Borg with systemd already. With some additional information mainly from the [Arch Wiki](https://wiki.archlinux.org/index.php/Systemd) and experimenting I found following setup to work for me.

So one part of the deal is the [timer](https://wiki.archlinux.org/index.php/Systemd/Timers) which is going to fire the service with the same name by default - `storagebox-borg.timer`:

```
[Unit]
Description=Borg User Backup Timer
 
[Timer]
WakeSystem=false
OnCalendar=hourly
RandomizedDelaySec=10min
Persistent=true
 
[Install]
WantedBy=timers.target

```

And the other part the service itself - `storagebox-borg.service`:

```
[Unit]
Description=Borg User Backup
 
[Service]
Type=simple
Wants=network-online.target
After=network-online.target
Nice=19
IOSchedulingClass=2
IOSchedulingPriority=7
Environment="BORG_RSH=ssh -i /root/.ssh/storagebox-from-root"
ExecStartPre=/usr/bin/borg break-lock ssh://USER@USER.your-storagebox.de:23/./backups/t480s
ExecStart=/home/jake/.local/bin/borg-backup.sh
```
 Both files have to be put into `/etc/systemd/system/`, where the custom systemwide services are.

Until now the `Wants` and `After` dependencies seem to work fine, but I didn't care enough yet to test if it works 100% as it would retry after about one hour anyway. Make sure to read the [notes](https://wiki.archlinux.org/index.php/Systemd#Running_services_after_the_network_is_up) on `network-online.target` as well.

This setup can be tested with `systemctl start storagebox-backup.service` while taking a look at the logs with `journalctl -u storagebox-backup -f`in another terminal window. If this throws no errors switch to production with enabling and starting the timer with `systemctl enable storagebox-backup.timer`, respective for `start`. Checking the timers works with `systemctl list-timers`.

## Access Test

To make sure there's access to the data - otherwise all the work would've been useless - try to mount the backup repo. I tried it as normal user, because I want to access as normal user and still had a valid user ssh key on the server. 
``export BORG_RSH='ssh -i /home/jake/.ssh/storagebox_backup'`` to use the user ssh key followed by
``borg mount ssh://USER@USER.your-storagebox.de:23/./backups/t480s /home/jake/MOUNTPOINT`` sould do the trick. That way I could verify added data was available in a later archive in the repo.

In the end unmount with ``borg umount /home/jake/MOUNTPOINT``.

## Further Notes

A tool like [borgmatic](https://github.com/witten/borgmatic) could help to achieve a more convenient Borg experience. But in my case everything should work as automated as possible anyway so working with bare Borg is just fine.

## Sources

Thanks:

- https://blog.andrewkeech.com/posts/170719_borg.html
- https://borgbackup.readthedocs.io/en/stable/quickstart.html#automating-backups (and the doc in general)
- https://wiki.hetzner.de/index.php/BorgBackup
- https://wiki.archlinux.org/index.php/Systemd (and subpages)
