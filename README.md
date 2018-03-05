## Upgrade from Aquamarine to Citrine

The procedure is quite as easy as minor version upgrade, but all components should be done in one shot. CERN advises to upgrade all servers at the same time (because of some changes in communication protocol) but partial upgrade works in test (safeguard by setting the instance read-only ?)

### Procedure

Run compaction for faster boot.

Stop all servers (FST + MGM), or decide to change servers one by one (by setting read-only on the whole instance to avoid errors)

**For safety, backup all `/var/eos` folders !!**

Change configuration (see [Configuration changes](#configuration-changes))

Change eos repos (see [yum.repo file](#yumrepo-file))

Run MGM upgrade (see [Upgrade MGM](#upgrade-mgm))

Run FST upgrade (see [Upgrade FSTs](#upgrade-fsts))

Some changes in using the instance (see [Changes in new version](#changes-in-new-version))

### Configuration changes

#### All servers

The eos configuration is moved from `/etc/sysconfig/eos` to `/etc/sysconfig/eos_env`. The `export` directive needs to be removed, as well as the service alias definition. Conversion to remove export directives seems to work this way :

    cat /etc/sysconfig/eos | sed -e 's/export //' > /etc/sysconfig/eos_env

If present, these lines need to be **removed** (they cause a warning in eos log at when starting) :

```
# ------------------------------------------------------------------
# Service Script aliasing for EL7 machines
# ------------------------------------------------------------------

which systemctl >& /dev/null
if [ $? -eq 0 ]; then
   alias service="service --skip-redirect"
fi
```

#### MGM

Necessary lines to be added in MGM's `/etc/xrd.cf.mgm`

```
#-------------------------------------------------------------------------------
# Set the namespace plugin implementation
#-------------------------------------------------------------------------------

mgmofs.nslib /usr/lib64/libEosNsInMemory.so
```

Now also in the configuration, synchronization service needs to explicitly define local and remote MGM host (**_! configuration is different on both MGMs_**) as `EOS_MGM_MASTER1/2` values are not sufficient any more. In `/etc/sysconfig/eos_env`, add :

```
EOS_MGM_HOST=fqdn.of.local.host.domain
EOS_MGM_HOST_TARGET=fqdn.of.remote.host.domain
```

#### FST

The FSTs need to be geotagged, otherwise no write can be scheduled there. So on all FSTs in `/etc/sysconfig/eos` add the value (can be anything to work, just not empty; can be set to rack or switch to foresee future expansions)

    export EOS_GEOTAG='JRC_DC'

### yum.repo file

https://eos-docs.web.cern.ch/eos-docs/quickstart/setup_repo.html#eos-citrine

`/etc/yum.repos.d/eos.repo`

```
[eos-citrine]
name=EOS 4.0 Version
baseurl=https://storage-ci.web.cern.ch/storage-ci/eos/citrine/tag/el-7/x86_64/
gpgcheck=0
enabled=1

[eos-dep]
name=EOS 4.0 Dependencies
baseurl=https://storage-ci.web.cern.ch/storage-ci/eos/citrine-depend/el-7/x86_64/
gpgcheck=0
enabled=1
```

(remove any obsolete `/etc/yum.repos.d/eos-dep.repo` if any)

Use also the xrootd-stable repository :

```
[xrootd-stable]
name=XRootD Stable repository
baseurl=http://xrootd.org/binaries/stable/slc/7/$basearch
gpgcheck=1
enabled=1
protect=0
gpgkey=http://xrootd.cern.ch/sw/releases/RPM-GPG-KEY.txt
```

### Upgrade MGM

    yum upgrade "eos-*"

Start eos, server should boot normally

Update also slave

Check synchronisation, etc...

### Upgrade FSTs

    yum upgrade "eos-*"

Expect a full boot because they need to fully resynchronize all FS with MGM (probably 30 min to 1 hour, depends on the number of files)

## Changes in new version

#### Service management systemd

The eos server now fully uses systemd, so systemctl command to handle services. Commands are :

`systemctl start eos` to start all eos services (including eossync)
`systemctl restart eos@*` to restart all eos services
`systemctl start eos@mgm` to start just mgm
`systemctl restart eossync@*` to restart eossync services
`systemctl status eossync@*` to get synchronization status (not as easily readable as old version)

Details are here http://eos-docs.web.cern.ch/eos-docs/eos_services.html

#### Master/slave switch

the previous command `service --skip-redirect eos master mq` doesn't work any more. The replacements are :

    systemctl start eos@master

    systemctl start eos@slave
