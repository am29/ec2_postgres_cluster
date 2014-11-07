# ansible_postgres_cluster

Proof-of-concept-quality Ansible tools for configuring a PostgreSQL
9.4 replication pair on EC2 using striped EBS volumes.

## Striped volume snapshots

Usage:

```yaml
  local_action: >
    ebs_batch_snapshot
	instance_id={{ ansible_ec2_instance_id }}
	volume_group={{ volume_group }}
	region={{ ansible_ec2_placement_region }}
```
