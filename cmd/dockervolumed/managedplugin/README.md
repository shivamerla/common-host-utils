
# Requirements

-   Docker Engine 17.09 or greater
-   If using Docker Enterprise Edition 2.x, the plugin is only supported in swarmmode
-   Recent Red Hat, Debian or Ubuntu-based Linux distribution
-   NimbleOS 5.0.8 or greater on a HPE Nimble Storage array


**Note:** Docker does not support certified and managed Docker Volume plugins with Kubnernetes. If you want to use Kubernetes on Docker with HPE Nimble Storage, please use the [HPE Flexvolume Plugins](https://infosight.hpe.com/tenant/Nimble.Tenant.0013400001Ug0UxAAJ/resources/nimble/software/Integration%20Kits/HPE%20Nimble%20Storage%20Linux%20Toolkit%20(NLT)) and follow the HPE Nimble Storage Integration Guide for Docker Enterprise Edition found on [HPE InfoSight](https://infosight.hpe.com) to deploy a fully supported solution.

# Limitations

HPE Nimble Storage provides a Docker certified plugin delivered through the Docker Store. HPE Nimble Storage also provides a Docker Volume plugin for Windows Containers as part of the Nimble Windows Toolkit (NWT) which is available on [HPE InfoSight](https://infosight.hpe.com/tenant/Nimble.Tenant.0013400001Ug0UxAAJ/resources/nimble/software/Integration%20Kits/HPE%20Nimble%20Storage%20Docker%20Volume%20Plugin). Certain features and capabilities are not available through the managed plugin. Please understand these limitations before deploying either of these plugins.

The managed plugin does NOT provide:

-   Support for Docker's release of Kubernetes in Docker Enterprise Edition 2.x
-   Locally scoped volumes
-   Support for older versions of NimbleOS (all versions below 5.x)
-   Support for Windows Containers

The managed plugin does provide a simple way to manage the HPE Nimble Storage integration on your Docker hosts using Docker's interface to install and manage the plugin.

# How to Use this Plugin

## Plugin Privileges

In order to create connections, attach devices and mount file systems, the plugin requires more privileges than a standard application container. These privileges are enumerated during installation. These permissions need to be granted for the plugin to operate correctly.

```
Plugin "nimble" is requesting the following privileges:
 - network: [host]
 - mount: [/dev]
 - mount: [/run/lock]
 - mount: [/sys]
 - mount: [/etc]
 - mount: [/var/lib]
 - mount: [/var/run/docker.sock]
 - mount: [/sbin/iscsiadm]
 - mount: [/lib/modules]
 - mount: [/usr/lib64]
 - allow-all-devices: [true]
 - capabilities: [CAP_SYS_ADMIN CAP_SYS_MODULE CAP_MKNOD]
```

## Host Configuration and Installation

Setting up the plugin varies between Linux distributions. The following workflows have been tested using a Nimble iSCSI group array at **192.168.171.74** with username **admin** and password **admin**:

These procedures **requires** root privileges.

Red Hat 7.5+, CentOS 7.5+, Oracle Enterprise Linux 7.5+ and Fedora 28+:

```
yum install -y iscsi-initiator-utils device-mapper-multipath
docker plugin install --disable --grant-all-permissions --alias nimble store/nimblestorage/nimble:2.5.1
docker plugin set nimble PROVIDER_IP=192.168.171.74 PROVIDER_USERNAME=admin PROVIDER_PASSWORD=admin
docker plugin enable nimble
systemctl daemon-reload
systemctl enable iscsid multipathd
systemctl start iscsid multipathd
```

Ubuntu 16.04 LTS and Ubuntu 18.04 LTS (Ubuntu 19.10 also tested):

```
apt-get install -y open-iscsi multipath-tools xfsprogs
modprobe xfs
sed -i"" -e "\$axfs" /etc/modules
docker plugin install --disable --grant-all-permissions --alias nimble store/nimblestorage/nimble:2.5.1
docker plugin set nimble PROVIDER_IP=192.168.171.74 PROVIDER_USERNAME=admin PROVIDER_PASSWORD=admin glibc_libs.source=/lib/x86_64-linux-gnu
docker plugin enable nimble
systemctl daemon-reload
systemctl restart open-iscsi multipath-tools
```

Debian 9.x (stable):

```
apt-get install -y open-iscsi multipath-tools xfsprogs
modprobe xfs
sed -i"" -e "\$axfs" /etc/modules
docker plugin install --disable --grant-all-permissions --alias nimble store/nimblestorage/nimble:2.5.1
docker plugin set nimble PROVIDER_IP=192.168.171.74 PROVIDER_USERNAME=admin PROVIDER_PASSWORD=admin iscsiadm.source=/usr/bin/iscsiadm glibc_libs.source=/lib/x86_64-linux-gnu
docker plugin enable nimble
systemctl daemon-reload
systemctl restart open-iscsi multipath-tools
```

### Making Changes

The `docker plugin set` command can only be used on the plugin if it is disabled. To disable the plugin, use the `docker plugin disable` command. For example:

```
$ docker plugin disable nimble
```

### Security Consideration

The HPE Nimble Storage Group credentials are visible to any user who can execute `docker plugin inspect nimble`. To limit credential visibility, the variables should be unset after certificates have been generated. The following set of steps can be used to accomplish this:

1.  Add the credentials
    -   `$ docker plugin set nimble ip=192.168.171.74 username=admin password=admin`
2.  Start the plugin
    -   `$ docker plugin enable nimble`
3.  Stop the plugin
    -   `$ docker plugin disable nimble`
4.  Remove the credentials
    -   `$ docker plugin set nimble username="" password=""`
5.  Start the plugin
    -   `$ docker plugin enable nimble`

**Note:** Certificates are stored in `/etc/nimblestorage` on the host and will be preserved across plugin updates.

In the event of reassociating the plugin with a different HPE Nimble Storage group, certain procedures need to be followed:

1.  Disable the plugin
    -   `$ docker plugin disable nimble`
2.  Set new paramters
    -   `$ docker plugin set nimble PROVIDER_REMOVE=true`
3.  Enable the plugin
    -   `$ docker plugin enable nimble`
4.  Disable the plugin
    -   `$ docker plugin disable nimble`
5.  The plugin is now ready for re-configuration
    -   `$ docker plugin set nimble ip=< New IP address > username=admin password=admin remove=false`

**Note:** The `remove=false` parameter must be set if the plugin ever has been unassociated from a HPE Nimble Storage group.

### Configuration Files and Options

The configuration directory for the plugin is `/etc/hpe-storage` on the host. Files in this directory are preserved between plugin upgrades. The `/etc/hpe-storage/volume-driver.json` file contains three sections, `global`, `defaults` and `overrides`. The global options are plugin runtime parameters and doesn't have any end-user configurable keys at this time.

The `defaults` map allows the docker host administrator to set default options during volume creation. The docker user may override these default options with their own values for a specific option.

The `overrides` map allows the docker host administrator to enforce a certain option for every volume creation. The docker user may not override the option and any attempt to do so will be silently ignored.

These maps are essential to discuss with the HPE Nimble Storage administrator. A common pattern is that a default protection template is selected for all volumes to fulfill a certain data protection policy enforced by the business it's serving. Another useful option is to override the volume placement options to allow a single HPE Nimble Storage array to provide multi-tenancy for docker environments.

**Note:** `defaults` and `overrides` are dynamically read during runtime while `global` changes require a plugin restart.

Below is an example `/etc/nimblestorage/volume-driver.json` outlining the above use cases:

```
{
  "global": {
    "nameSuffix": ".docker"
  },
  "defaults": {
    "description": "Volume provisioned by Docker",
    "protectionTemplate": "Retain-90Daily"
  },
  "overrides": {
    "folder": "docker-prod"
  }
}
```

For an exhaustive list of options, either refer to the [HPE Nimble Storage Linux Toolkit](https://infosight.hpe.com/tenant/Nimble.Tenant.0013400001Ug0UxAAJ/resources/nimble/software/Integration%20Kits/HPE%20Nimble%20Storage%20Linux%20Toolkit%20(NLT)) documentation or use the `help` option from the docker CLI:

```
$ docker volume create -d nimble -o help
Nimble Storage Docker Volume Driver: Create Help
Create or Clone a Nimble Storage backed Docker Volume or Import an existing Nimble Volume or Clone of a Snapshot into Docker.

Universal options:
  -o mountConflictDelay=X X is the number of seconds to delay a mount request when there is a conflict (default is 0)

Create options:
  -o sizeInGiB=X          X is the size of volume specified in GiB
  -o size=X               X is the size of volume specified in GiB (short form of sizeInGiB)
  -o fsOwner=X            X is the user id and group id that should own the root directory of the filesystem, in the form of [userId:groupId]
  -o fsMode=X             X is 1 to 4 octal digits that represent the file mode to be applied to the root directory of the filesystem
  -o description=X        X is the text to be added to volume description (optional)
  -o perfPolicy=X         X is the name of the performance policy (optional)
                          Performance Policies: Exchange 2003 data store, Exchange 2007 data store, Exchange log, SQL Server, SharePoint, 
                          Exchange 2010 data store, SQL Server Logs, SQL Server 2012, Oracle OLTP, Windows File Server, Other Workloads, Backup Repository, 
                          Veeam Backup Repository
  -o pool=X               X is the name of pool in which to place the volume (optional)
  -o folder=X             X is the name of folder in which to place the volume (optional)
  -o encryption           indicates that the volume should be encrypted (optional, dedupe and encryption are mutually exclusive)
  -o thick                indicates that the volume should be thick provisioned (optional, dedupe and thick are mutually exclusive)
  -o dedupe               indicates that the volume should be deduplicated
  -o limitIOPS=X          X is the IOPS limit of the volume. IOPS limit should be in range [256, 4294967294] or -1 for unlimited.
  -o limitMBPS=X          X is the MB/s throughput limit for this volume. If both limitIOPS and limitMBPS are specified, limitMBPS must not be hit before limitIOPS
  -o destroyOnRm          indicates that the Nimble volume (including snapshots) backing this volume should be destroyed when this volume is deleted
  -o protectionTemplate=X X is the name of the protection template (optional)
                          Protection Templates: Retain-30Daily, Retain-90Daily, Retain-48Hourly-30Daily-52Weekly

Clone options:
  -o cloneOf=X            X is the name of Docker Volume to create a clone of
  -o snapshot=X           X is the name of the snapshot to base the clone on (optional, if missing, a new snapshot is created)
  -o createSnapshot       indicates that a new snapshot of the volume should be taken and used for the clone (optional)
  -o destroyOnRm          indicates that the Nimble volume (including snapshots) backing this volume should be destroyed when this volume is deleted
  -o destroyOnDetach      indicates that the Nimble volume (including snapshots) backing this volume should be destroyed when this volume is unmounted or detached

Import Volume options:
  -o importVol=X          X is the name of the Nimble Volume to import
  -o pool=X               X is the name of the pool in which the volume to be imported resides (optional)
  -o folder=X             X is the name of the folder in which the volume to be imported resides (optional)
  -o forceImport          forces the import of the volume.  Note that overwrites application metadata (optional)
  -o restore              restores the volume to the last snapshot taken on the volume (optional)
  -o snapshot=X           X is the name of the snapshot which the volume will be restored to, only used with -o restore (optional)
  -o takeover             indicates the current group will takeover the ownership of the Nimble volume and volume collection (optional)
  -o reverseRepl          reverses the replication direction so that writes to the Nimble volume are replicated back to the group where it was replicated from (optional)

Import Clone of Snapshot options:
  -o importVolAsClone=X   X is the name of the Nimble Volume and Nimble Snapshot to clone and import
  -o snapshot=X           X is the name of the Nimble snapshot to clone and import (optional, if missing, will use the most recent snapshot)
  -o createSnapshot       indicates that a new snapshot of the volume should be taken and used for the clone (optional)
  -o pool=X               X is the name of the pool in which the volume to be imported resides (optional)
  -o folder=X             X is the name of the folder in which the volume to be imported resides (optional)
  -o destroyOnRm          indicates that the Nimble volume (including snapshots) backing this volume should be destroyed when this volume is deleted
  -o destroyOnDetach      indicates that the Nimble volume (including snapshots) backing this volume should be destroyed when this volume is unmounted or detached
```

## Use

The [HPE Nimble Storage Linux Toolkit](https://infosight.hpe.com/tenant/Nimble.Tenant.0013400001Ug0UxAAJ/resources/nimble/software/Integration%20Kits/HPE%20Nimble%20Storage%20Linux%20Toolkit%20(NLT)) documentation covers basic usage of the plugin. The following example demonstrates the `create` command with a set of options:

```
$ docker volume create -d nimble -o size=300 -o description="Example" -o perfPolicy="Oracle OLTP" -o encryption="true" -o dedupe="true" -o protectionTemplate="Retain-30Daily" --name example
```

## Uninstall

The plugin can be removed using the `docker plugin rm` command. This command will not remove the configuration directory (/etc/nimblestorage).

```
$ docker plugin rm nimble
```

## Debugging

The [HPE Nimble Storage Linux Toolkit](https://infosight.hpe.com/tenant/Nimble.Tenant.0013400001Ug0UxAAJ/resources/nimble/software/Integration%20Kits/HPE%20Nimble%20Storage%20Linux%20Toolkit%20(NLT)) documentation covers basic debugging. That documentation applies to this plugin as well.  Plugin logs will be under`/var/log`
