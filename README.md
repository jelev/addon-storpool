# StorPool Storage Driver

## Description

The [StorPool](https://storpool.com/) datastore driver enables OpenNebula to use a StorPool storage system for storing disk images.

## Development

To contribute bug patches or new features, you can use the github Pull Request model. It is assumed that code and documentation are contributed under the Apache License 2.0. 

More info:

* [How to Contribute](http://opennebula.org/addons/contribute/)
* Support: [OpenNebula user forum](https://forum.opennebula.org/c/support)
* Development: [OpenNebula developers forum](https://forum.opennebula.org/c/development)
* Issues Tracking: GitHub issues (https://github.com/OpenNebula/addon-storpool/issues)

## Authors

* Leader: Anton Todorov (a.todorov@storpool.com)

## Compatibility

This add-on is compatible with OpenNebula 4.10, 4.12 and StorPool 15.02+.

## Requirements

### OpenNebula Front-end

Password-less SSH access from the front-end `oneadmin` user to the `node` instances.

### OpenNebula Node

* The OpenNebula admin account `oneadmin` must be member of the 'disk' system group to have access to the StorPool block device during image create/import operations.
* StorPool initiator driver (storpool_block)
* StorPool CLI, API access and token

### StorPool cluster

A working StorPool cluster is required.

## Limitations

1. compatible only with the KVM hypervisor
1. no support for VM snapshot because it is handled internally by libvirt
1. no support volatile disks
1. imported images are thick provisioned


## Installation

* Clone the addon-storpool
```bash
cd /usr/src
git clone https://github.com/OpenNebula/addon-storpool
```
* Run the install script and chek for any reported errors or warnings
```bash
bash addon-storpool/install.sh
```
* Restart `opennebula` and `opennebula-sunstone` services

## Configuration

### Configuring the System Datastore

The system datastore must be configured with transfer manager (TM_MAD) backend of type shared or ssh. This system datastore will hold only the symbolic links to the StorPool block devices and context configuration, so it will not take much space. See more details on the [System Datastore Guide](http://docs.opennebula.org/4.10/administration/storage/system_ds.html).

### Configuring the Image Datastore

Some configuration attributes must be set to enable an image datastore as StorPool enabled one:

* **DS_MAD**: [mandatory] The DS driver for the datastore. String, use value `storpool`
* **TM_MAD**: [mandatory] Transfer driver for the datastore. String, use value `storpool`
* **DISK_TYPE**: [mandatory] Type for the VM disks using images from this datastore. String, use value `block`
* **BRIDGE_LIST**: [mandatory] Nodes to use for image datastore operations. String (1)
* **SP_REPLICATION**: [mandatory] The StorPool replication level for the datastore. Number (2)
* **SP_PLACEALL**: [mandatory] The name of StorPool placement group of disks where to store data. String (3)
* **SP_PLACETAIL**: [optional] The name of StorPool placement group of disks from where to read data. String (4)


1. Quoted, space separated list of server hostnames which are members of the StorPool cluster.
1. The replication level defines how many separate copies to keep for each data block. Supported values are: `1`, `2` and `3`.
1. The PlaceAll placement group is defined in StorPool as list of drives where to store the data.
1. The PlaceTail placement group is defined in StorPool as list of drives. used in StorPool hybrid setup. If the setup is not of hybrid type leave blank or same as **SP_PLACEALL**

The following example illustrates the creation of a StorPool datastore of hybrid type with 3 replicas. In this case the datastore will use hosts node1, node2 and node3 for imports and creating images.

#### Through Sunstone

Sunstone -> Infrastructure -> Datastores -> Add [+]

* Name: StorPool
* Presets: StorPool
* Host Bridge List: node1 node2 node3
* StorPool Replication: 3
* StorPool PlaceAll: hdd
* StorPool PlaceTail: ssd

#### Through onedatastore

```bash
# create datastore configuration file
$ cat >/tmp/ds.conf <<EOF
NAME = "StorPool"
DS_MAD = "storpool"
TM_MAD = "storpool"
DISK_TYPE = "block"
BRIDGE_LIST = "node1 node2 node3"
SP_REPLICATION = 3
SP_PLACEALL = "hdd"
SP_PLACETAIL = "ssd"
EOF

# Create datastore
$ onedatastore create /tmp/ds.conf

# Verify datastore is created
$ onedatastore list

  ID NAME                SIZE AVAIL CLUSTER      IMAGES TYPE DS       TM
   0 system             98.3G 93%   -                 0 sys  -        shared
   1 default            98.3G 93%   -                 0 img  fs       shared
   2 files              98.3G 93%   -                 0 fil  fs       ssh
 100 StorPool            2.4T 99%   -                 0 img  storpool storpool
```

Note that by default OpenNebula assumes that the Datastore is accessible by all hypervisors and all bridges. If you need to configure datastores for just a subset of the hosts check the [Cluster guide](http://opennebula.org/documentation:rel4.4:cluster_guide).

### Usage

Once configured, the StorPool image datastore can be used as a backend for disk images.

Non-persistent images are StorPool volumes. When you use a non-persistent image for a VM the driver creates a new temporary volume and attaches the new volume to the VM. When the VM is destroyed, the temporary volume is deleted.
