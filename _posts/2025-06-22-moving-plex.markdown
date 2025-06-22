---
layout: post
title:  "Migrating Plex from TrueNAS CORE (fka FreeNAS) to TrueNAS SCALE"
date: 2025-05-30 10:30:00 -0700
tags: computers freenas truenas plex
categories: linux
---

# Notes on migrating Plex

The Plex app that TrueNAS CORE managed is dead. A moment of silence, please. The jail-based App system in TrueNAS CORE was perfect for BSD diehards, but containers have won out, and TrueNAS SCALE's app system is based (as of versions 24 and 25) on Docker.

This is nice, because you can get Plex updates faster, and it opens up some more capabilities that I'm excited about, but this is mostly documenting how I moved my Plex install over with minimal downtime.


## Migrating from CORE to SCALE:
* You have to enable SSH.
* There's no more root user.


# Plex documentation
Plex has [written up a guide](https://support.plex.tv/articles/201370363-move-an-install-to-another-system/) that is partially helpful in migrating a server. You need to move the following directories:
* Cache
* Diagnostics
* Media* Metadata
* Plug-in Support
* Preferences.xml

To do that: 
1. Log in to Plex, 'Disable Emptying of Trash'
2. `ssh` to your old FreeNAS system, then build a tarball of the important files.
3. `cd '/mnt/your_pool_here/iocage/jails/plex-plexpass/root/Plex Media Server'`
4. `tar zcf /mnt/your_pool_here/Bulk\ Storage/plex-backup.tgz Cache Diagnostics Media Metadata "Plug-in Support" Preferences.xml`
    * Note I had a dataset called "Bulk Storage" which was a useful temporary storage for this blob of data.
5. ...wait...
    * Keep waiting
6. Probably because I had [this option](https://forums.plex.tv/t/plex-media-server-media-folder-size/54704/2) enabled, so there's a lot of thumbnails. 10-50MB per item in the library.
    * Also note no `v`; this is on purpose. I want CPU cycles spent *compressing* not *outputing a wall of text*. Although, since I was probably limited more by disk IO than CPU, maybe this doesn't matter all that much.
    * In any case, you can use `^T` to get a status report:
```
load: 0.86  cmd: bsdtar 3635 [running] 1580.86r 977.43u 36.49s 69% 9980k
dbuf_read+0x84d dmu_buf_will_dirty_impl+0x11e dmu_write_uio_dnode+0xc9 dmu_write_uio_dbuf+0x42 zfs_write+0x5e4 zfs_freebsd_write+0x44 VOP_WRITE_APV+0x99 vn_write+0x28b vn_io_fault_doio+0x43 vn_io_fault1+0x15c vn_io_fault+0x1b0 dofilewrite+0x88 sys_write+0xbc amd64_syscall+0x10c fast_syscall_common+0xf8
In: 55757 files, 33307801600 bytes; Out: 18291732480 bytes, compression 45%
Current: Media/localhost/5/900ff4c6159bcec7affebd3eec6f0d1d85f305a.bundle/Contents/Indexes/index-sd.bif (786432/9623349 bytes)
```
# Install the Plex app
On your new TrueNAS SCALE system, install the Plex app. Make sure to define config and data directories. I have an all-SSD pool for apps (3x NVME SSDs in a RAIDZ1 configuration). For the random IO that applications do, this will be faster than spinning rust, although with enough RAM ZFS will do a pretty good job of protecting you from the sins of your hardware.

In any case, I have a `/mnt/NASApps/plex` directory in which I have a `config` directory that is mounted to be the container's configuration. Following the guidance from Plex above, start and claim the server, shut it down, and then extract the tarball of your old Plex server. I extracted to a temporary directory (_not_ `/tmp`; it isn't big enough) and then moved files over.

⚠️⚠️⚠️  
When installing the App, in the "Network Configuration" section, check the "Host network" box! Otherwise all play will be indirect!  
⚠️⚠️⚠️


```
# cd /mnt/NASApps/temp_plex_work
# tar zxf plex-backup.tgz
# ls
 Cache	 Diagnostics   Media   Metadata  'Plug-in Support'   Preferences.xml   plex-backup.tgz
# ls /mnt/NASApps//plex/config/Library/Application\ Support/Plex\ Media\ Server
 Cache	 'Crash Reports'   Drivers   Media     'Plug-in Support'   Preferences.xml    Updates
 Codecs   Diagnostics	   Logs      Metadata   Plug-ins	  'Setup Plex.html'   plexmediaserver.pid
```

You can see the extracted files match to the existing files in the new server. Next is to copy our server's ~~brains~~ configurations over.

```
# cp -r /mnt/NASApps/temp_plex_work /mnt/NASApps/plex/config/Library/Application\ Support/Plex\ Media\ Server
# ls -lgh plex/config/Library/Application\ Support/Plex\ Media\ Server
total 83K
drwxr-xr-x 6 apps   20 Mar 20 15:05  Cache
drwxr-xr-x 3 apps    4 Mar 25 16:37  Codecs
drwxr-xr-x 4 apps    5 Mar 25 16:37 'Crash Reports'
drwxr-xr-x 2  972    2 Sep 19  2019  Diagnostics
drwxr-xr-x 2 apps    2 Mar 25 16:31  Drivers
drwxr-xr-x 2 root    2 Mar 25 16:31  Logs
drwxr-xr-x 3  972    3 Sep 18  2019  Media
drwxr-xr-x 4  972    4 Sep 18  2019  Metadata
drwxr-xr-x 7  972    7 Sep 17  2019 'Plug-in Support'
drwxr-xr-x 2 apps    2 Mar 25 16:31  Plug-ins
-rw------- 1  972 1.2K Mar 20 15:21  Preferences.xml
-rw------- 1 apps  13K Mar 25 16:37 'Setup Plex.html'
drwxr-xr-x 2 apps    2 Mar 25 16:31  Updates
-rw-r--r-- 1 apps    3 Mar 25 16:37  plexmediaserver.pid
```

Since the old files retained their permissions, you'll have to `chown -R apps:` everything so the UID matches. I don't know how much it matters, since most containers love to run as root and can totally ignore such pesky things as "permissions" in an effort to work everywhere.

Anyway. Start the container and connect to Plex in a web browser. Don't panic if you see this message:

```xml
This XML file does not appear to have any style information associated with it. The document tree is shown below.
<Response code="503" title="Maintenance" status="Plex Media Server is currently running database migrations."/>
```

Wait it out. Once Plex takes care of its housekeeping, you should be all set.

I also configured a cron job to fix up the permissions of media files that are copied over. I don't know for certain if this is required, or is due to how I migrated things over from the old system. But in any case, every few minutes I have these commands run:

```shell
    /bin/chmod -R 755 /mnt/NASPool/Media/
    /bin/chown -R "1000:1000" /mnt/NASPool/Media/
```




UPS configuration
https://www.wundertech.net/how-to-set-up-truenas-as-a-nut-server/


Removing GELI
1. zpool status
1. zpool offline POOL_NAME gptid/12345xxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.eli
1. geli detach gptid/12345xxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.eli
1. Wait for the detach to complete
1. zpool replace POOL_NAME new-id-number original-gptid-without-eli