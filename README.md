## Upgrade from Aquamarine to Citrine

Quite easy procedure. Need to upgrade all servers at the same time (because of change in communication protocol).
_Is there now a way to upgrade progressively, with limited downtime ?_

### Procedure

Run compaction for faster boot.

Stop all servers (FST + MGM).

For safety, backup all `/var/eos` folders.

Change configuration (see [#configuration changes](#configuration changes))

Change eos repos (see [#yum.repo file](#yum.repo file))

Run MGM upgrade (see [#Upgrade MGMs](#Upgrade MGMs))

Run FST upgrade (see [#Upgrade FSTs](#Upgrade FSTs))

Enjoy (or cry ?)

### configuration changes

Do we need to convert our `/etc/sysconfig/eos` files into `/etc/sysconfig/eos-env` ?

##### MGM
Necessary line to be added in file `/etc/xrd.cf.mgm`

```
#-------------------------------------------------------------------------------
# Set the namespace plugin implementation
#-------------------------------------------------------------------------------
mgmofs.nslib /usr/lib64/libEosNsInMemory.so
```

##### FST
The FSTs need to be geotagged, otherwise no write can be scheduled there. So on all FSTs in `/etc/sysconfig/eos` add the value (can be anything to work, just not empty; can be set to rack or switch to foresee future expansions)

    export EOS_GEOTAG='JRC_DC'

### yum.repo file

`/etc/yum.repos.d/eos.repo` sams as clients with new 4.1.25 version

```
[eos-citrine]
name=EOS 4.0 Version
baseurl=http://storage-ci.web.cern.ch/storage-ci/eos/citrine/tag/el-7/x86_64/
gpgcheck=0
enabled=1

[eos-dep]
name=EOS 4.0 Dependencies
baseurl=http://storage-ci.web.cern.ch/storage-ci/eos/citrine-depend/el-7/x86_64/
gpgcheck=0
enabled=1
```

Think also to re-activate stock epel xrootd package in `/etc/yum.repos.d/epel.repo` :

```
exclude=libmicrohttpd*
#exclude=xrootd*,libmicrohttpd*
```

### Upgrade MGMs

``yum upgrade eos-*``

Start eos, server should boot normally.

Update also the slave.

Check synchronisation, etc...

### Upgrade FSTs

``yum upgrade eos-*``

Expect a full boot because they need to fully resynchronise all FS with MGM (probably 30 min to 1 hour, depends on the number of files)
