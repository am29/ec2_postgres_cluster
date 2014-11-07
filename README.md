# ec2_postgres_cluster

PostgreSQL 9.4 on CentOS 7 on EC2 using striped EBS volumes. (Largely proof-of-concept-quality.)

Major points of interest:

## Striped volumes and snapshots

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

Both internal and external zones must already exist.

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
