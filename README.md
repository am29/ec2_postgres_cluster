# ec2_postgres_cluster

PostgreSQL 9.4 on CentOS 7 on EC2 using striped EBS volumes. (Largely proof-of-concept-quality.)

## Playbooks and inventory

`inventory` is a dynamic inventory file that groups hosts by the value of their `group` tag.

- `network.yml` creates or updates the CloudFormation template in
  `files/template.json`. It defines a VPC with two subnets, one
  network ACL and one security group.

- `create_instances.yml` creates two EC2 instances, one in each
  subnet, in the groups `pg_master` and `pg_replica`, respectively.

- `provision_master.yml` creates and starts a new PostgreSQL master
  with one replication slot. PGDATA is located on an EBS-backed RAID 0
  array.

- `snapshot_master.yml` snapshots the PostgreSQL database by briefly
  freezing the logical volumes, requesting snapshots of the underlying
  EBS volumes, and then unfreezing them.

- `provision_replica.yml` creates and starts a new PostgreSQL replica
  given the volume_group and timestamp of the snapshot set.

## Striped volumes and snapshots

This is the most useful part. `volume_group` is both the name of the LVM
group and the tag key to identify volumes and snapshots. Setting
`warm` to yes will
[pre-warm](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-prewarm.html)
the array, either by zeroing it out (in `create_array.yml`) or reading
every block (in `restore_array.yml`). Only tested with XFS on CentOS
7, although there's nothing too special there.

It's possible this (at least create/restore) would work better as an
Ansible role.

### Array creation

```yaml
- include: tasks/create_array.yml
  vars:
    volume_group: pg_master
    mdlevel: raid0
	mddev: /dev/md0
    devices: [ /dev/xvdh, /dev/xvdi, /dev/xvdj, /dev/xvdk ]
    size: 5
    piops: 100
    warm: no
	logical_volumes:
	  - name: data
        size: 90%VG
        mount: /mnt/data
        filesystem: xfs
      - name: xlog
        size: 5%VG
        mount: /mnt/xlog
        filesystem: xfs
      - name: log
        size: 5%VG
        mount: /mnt/log
        filesystem: xfs
```

### Array snapshot

```yaml
- include: tasks/snapshot.yml
  vars:
    volume_group: pg_master
    mounts:
      - /mnt/data
      - /mnt/xlog
      - /mnt/log
```
↓

![AWS Console](http://i.imgur.com/hmD0S0p.png)

### Restoring an array from a snapshot set

```yaml
- include: tasks/restore_array.yml
  vars:
    volume_group: pg_replica
    snapshot_volume_group: pg_master
    # Must be quoted, otherwise it will be interpreted as a native
    # time obj
    snapshot_timestamp: "2014-11-06T22:03:44Z"
    piops: 100
    warm: no
```

## Split-horizon A records for internal/external Route 53 zone pairs

Quick hack, since there's no way to specify an internal zone in the
core `route53` module. Both internal and external zones must already
exist.

```yaml
- local_action: >
    r53_split private_ip={{ ansible_ec2_local_ipv4 }}
	public_ip={{ ansible_ec2_public_ipv4 }}
	zone=aws.willstclair.net.
	hostname=db1.aws.willstclair.net
	state=present
```

## Module for PostgreSQL replication slots (9.4 and up)

Must run locally as the `postgres` user, using peer authentication.

```yaml
- pg_replication_slot: name=replica0 state=present
  sudo_user: postgres
```
