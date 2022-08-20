[![GitHub release](https://img.shields.io/github/release/Antynea/grub-btrfs.svg)](https://github.com/Antynea/grub-btrfs/releases)
![](https://img.shields.io/github/license/Antynea/grub-btrfs.svg)

## grub-btrfs 

* BTC donation address: `1Lbvz244WA8xbpHek9W2Y12cakM6rDe5Rt`
- - -
### Description
Improves grub by adding "btrfs snapshots" to the grub menu.

You can boot your system on a "snapshot" from the grub menu.  
Supports manual snapshots, snapper, timeshift ...

##### Warning: booting on read-only snapshots can be tricky

If you choose to do it, `/var/log` or even `/var` must be on a separate subvolume.  
Otherwise, make sure your snapshots are writeable.  
See [this ticket](https://github.com/Antynea/grub-btrfs/issues/92) for more info.

This project includes its own solution.  
Refer to the [documentation](https://github.com/Antynea/grub-btrfs/blob/master/initramfs/readme.md).

- - -
### What features does grub-btrfs have?
* Automatically list snapshots existing on root partition (btrfs).
* Automatically detect if `/boot` is in separate partition.
* Automatically detect kernel, initramfs and intel/amd microcode in `/boot` directory on snapshots.
* Automatically create corresponding "menuentry" in `grub.cfg`
* Automatically detect the type/tags and descriptions/comments of snapper/timeshift snapshots.
* Automatically generate `grub.cfg` if you use the provided systemd service.

- - -
### Installation
#### Arch Linux
The package is available in the community repository [grub-btrfs](https://archlinux.org/packages/community/any/grub-btrfs/)
```
pacman -S grub-btrfs
```

#### Gentoo
grub-btrfs is only available in the Gentoo User Repository (GURU) and not in the official Gentoo repository.  
If you have not activated the GURU yet, do so by running:
```
emerge -av app-eselect/eselect-repository 
eselect repository enable guru 
emerge --sync 
```
If you are using SystemD on Gentoo, make sure the USE-Flag `systemD` is set. (Either globally in make.conf or in package.use for the package app-backup/grub-btrfs)
Without systemD USE-Flag the OpenRC-Daemon of grub-btrfs will be installed.

Emerge grub-btrfs via 
`emerge app-backup/grub-btrfs`

#### Kali Linux
[grub-btrfs](http://pkg.kali.org/pkg/grub-btrfs) is available in the Kali Linux repository and can be installed with:  
```
apt install grub-btrfs
```
Booting into read-only snapshots is fully supported when choosing "btrfs" as file system during a standard Kali Linux installation following [this walk-through](https://www.kali.org/docs/installation/btrfs/).  

#### Manual
* Run `make install` or look into Makefile for instructions on where to put each file.
* Run `make help` to check what options are available. 
* Dependencies:
  * [btrfs-progs](https://archlinux.org/packages/core/x86_64/btrfs-progs/)
  * [grub](https://archlinux.org/packages/core/x86_64/grub/)
  * [bash >4](https://archlinux.org/packages/core/x86_64/bash/)
  * [gawk](https://archlinux.org/packages/core/x86_64/gawk/)

#### NOTE: All distros
Generate your grub menu after installation for the changes to take effect.  
For example:  
On **Arch Linux** or **Gentoo** use `grub-mkconfig -o /boot/grub/grub.cfg`.  
On **Fedora** use `grub2-mkconfig -o /boot/grub2/grub.cfg`  
On **Debian-like** distribution `update-grub` is an alias to `grub-mkconfig ...`
- - -
### Customization
You have the possibility to modify many parameters in `/etc/default/grub-btrfs/config`.  
See [config file](https://github.com/Antynea/grub-btrfs/blob/master/config) for more information.

#### Warning:
by default, `grub-mkconfig` command is used.  
Might be `grub2-mkconfig` on some systems (Fedora ...).   
Edit `GRUB_BTRFS_MKCONFIG` variable in `/etc/default/grub-btrfs/config` file to reflect this.
- - -
### SystemD service
#### Automatically update grub upon snapshot
To automatically regenerate `grub-btrfs.cfg` when a modification appears in the `/.snapshots` mount point, run
```
systemctl enable grub-btrfs.path
systemctl start grub-btrfs.path  # In case the mount point is available already
```
Monitoring starts automatically when the mount point becomes available.
    
##### Snapshots not in `/.snapshots`
To modify `grub-btrfs.path` run
```
systemctl edit --full grub-btrfs.path
systemctl reenable grub-btrfs.path
```
To find out the name of the `.mount` unit use `systemctl list-units -t mount`.

#### Automatically update grub upon restart/ boot:
[Look at this comment](https://github.com/Antynea/grub-btrfs/issues/138#issuecomment-766918328)  
Currently not implemented

**Timeshift**

1. Run `systemctl edit --full grub-btrfs.path`
1. Replace the whole block by:
```
[Unit]
Description=Monitors for new snapshots
DefaultDependencies=no
Requires=run-timeshift-backup.mount
After=run-timeshift-backup.mount
BindsTo=run-timeshift-backup.mount

[Path]
PathModified=/run/timeshift/backup/timeshift-btrfs/snapshots

[Install]
WantedBy=run-timeshift-backup.mount
```
1. Run `systemctl reenable grub-btrfs.path` to reload the changes you made

1. Run `systemctl start grub-btrfs.path` to start monitoring.<br>Otherwise, the unit will automatically start monitoring when the mount point will be available.  

Note:
You can view your change to `systemctl cat grub-btrfs.path`.
To revert change use `systemctl revert grub-btrfs.path`.

Note: If you are using timeshift version v22.06 or newer, snapshots are now saved in the direcory `/run/timeshift/$PIDofthecurrentlyrunnigtimeshift/backup/timeshift-btrfs` by Timeshift. The PID is changing everytime you open Timeshift. This can not be changed, but when the mount `/run/timeshift/backup/` is created in `/etc/fstab` timeshift will use that. For further information on this workaround see [Lorenzo Bettinis blog](https://www.lorenzobettini.it/2022/07/timeshift-and-grub-btrfs-in-linux-arch/).

##
### OpenRC daemon
#### Automatically update grub upon snapshot
If you would like grub-btrfs menu to automatically update when a snapshot is created or deleted, configure `/etc/conf.d/grub-btrfsd` and after that start the daemon.
###### Timeshift versions >= v22.06 (default)
This is the default setting, you don't need to change anything in `/etc/conf.d/grub-btrfsd`, keep on reading at the "How to (auto)start the daemon" heading. 
Newer versions of Timeshift mount their snapshots to `/run/timeshift/$PIDofthecurrentlyrunnigtimeshift/backup/timeshift-btrfs`. The PID is changing everytime you close Timeshift and open it again. The OpenRC-Daemon can automatically take care of the detection of the correct PID and directory if you set the variable `timeshift_auto` to `true` in `/etc/conf.d/grub-btrfsd`. In this case the variable `snapshots` has no influence.
###### Timeshift versions < v22.06
If you don't want to use `timeshift_auto`, set the variable `snapshots` in the file `/etc/conf.d/grub-btrfsd` to the path of your snapshot directory. This is usually `/run/timeshift/backup/timeshift-btrfs/snapshots`. This will not work for newer Timeshift versions, however there is a [workaround](https://www.lorenzobettini.it/2022/07/timeshift-and-grub-btrfs-in-linux-arch/) that works for now. Basically you need to mount the root subvolume to `/run/timeshift/backup` by putting this line into `/etc/fstab`:
```
UUID=<UUID> /run/timeshift/backup       btrfs   defaults,noatime  0 0
```
You can get your UUID with the command `blkid`.

###### Snapper
For snapper set `timeshift-auto=false` and `snapshots` to Snapper snapshots directory. (usually `/.snapshots`)
##### How to (auto)start the daemon
Use `sudo rc-config add grub-btrfsd default`, to start the grub-btrfsd daemon the next time the system boots. 
To start `grub-btrfsd` right now, run `sudo rc-service grub-btrfsd start`.
`grub-btrfsd` then automatically watches the snapshot directory and updates the grub-menu when a change occurs.
When you changed a setting in `/etc/conf.d/grub-btrfd`, you have to restart the daemon with `sudo rc-service grub-btrfd restart` to make the change become effective. 

#### Automatically update grub upon restart/boot:
If you would like the grub-btrfs menu to automatically update on system restart/ shutdown, just add the following script as `/etc/local.d/grub-btrfs-update.stop`:
```bash
#!/usr/bin/env bash

description="Update the grub btrfs snapshots menu"
name="grub-btrfs-update"

depend()
{
	use localmount
}
	
bash -c 'if [ -s "${GRUB_BTRFS_GRUB_DIRNAME:-/boot/grub}/grub-btrfs.cfg" ]; then /etc/grub.d/41_snapshots-btrfs; else {GRUB_BTRFS_MKCONFIG:-grub-mkconfig} -o {GRUB_BTRFS_GRUB_DIRNAME:-/boot/grub}/grub.cfg; fi' 
```

Make your script executable with `chmod a+x /etc/local.d/grub-btrfs-update.stop`.

* The extension ".stop" at the end of the filename indicates to locald that this script should be run at shutdown. 
 If you want to run the menu update on startup instead, rename the file to `grub-btrfs-update.start`
* Works for snapper and timeshift

- - -
### Special thanks for assistance and contributions
* [Maxim Baz](https://github.com/maximbaz)
* [Schievel1](https://github.com/Antynea/grub-btrfs/discussions/173#discussioncomment-1438790)
* [All contributors](https://github.com/Antynea/grub-btrfs/graphs/contributors)
- - -
