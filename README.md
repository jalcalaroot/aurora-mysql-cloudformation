# Aurora MySQL Cluster (CloudFormation)

CloudFormation template that deploys a provisioned Amazon Aurora MySQL cluster with two instances in an existing VPC, with the master credentials generated and stored in AWS Secrets Manager. Parameter labels and descriptions in the template are in Spanish.

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

Template: `aurora-mysql-cloudformation.yml`

- Secrets Manager secret with a generated 32-character password (username from `DBMasterUsername`), attached to the cluster
- Security group allowing TCP 3306 from `DBPermitLocation`
- DB subnet group from the selected subnets
- Cluster parameter group (utf8 / `utf8_general_ci`) and an empty instance parameter group
- Aurora MySQL cluster (`provisioned` mode, storage encrypted, `DeletionPolicy: Snapshot`)
- Two DB instances (writer + reader), `PubliclyAccessible: false`
- Outputs/exports: cluster name, writer/reader endpoints, port, security group ID, secret ARN

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `VpcIdParameter` | Existing VPC | (required) |
| `DBSubnets` | Subnets for the DB subnet group | (required) |
| `DBClusterIdentifierName` | Cluster identifier | `mi-cluster1` |
| `DBEngineVersion` | `5.7.mysql-aurora.2.10.2` or `8.0.mysql-aurora.3.02.0` | `8.0.mysql-aurora.3.02.0` |
| `DBInstanceClass` | Instance class (t3/t4g/r5/r6g) | `db.t3.medium` |
| `DBPermitLocation` | CIDR allowed to reach port 3306 | `0.0.0.0/0` |
| `DBName` | Initial database name | `mibd` |
| `DBSecretName` | Secrets Manager secret name | `<SECRET_NAME>` |
| `DBMasterUsername` | Master username | `admin` |
| `DBPreferredBackupWindow` | Daily backup window (UTC) | `05:20-05:55` |
| `DBBackupRetentionPeriod` | Backup retention days (1-35) | `7` |
| `DBPreferredMaintenanceWindow` | Weekly maintenance window (UTC) | `Sat:07:00-Sat:07:30` |
| `DBAutoMinorVersionUpgrade` | Auto minor version upgrade | `true` |
| `DBDeletionProtection` | Cluster deletion protection | `true` |

## Deploy

```bash
aws cloudformation deploy \
  --stack-name <STACK_NAME> \
  --template-file aurora-mysql-cloudformation.yml \
  --parameter-overrides \
    VpcIdParameter=<VPC_ID> \
    DBSubnets=<SUBNET_ID_1>,<SUBNET_ID_2> \
    DBPermitLocation=<APP_CIDR> \
    DBSecretName=<SECRET_NAME>
```

No IAM capabilities are required. The listed engine versions are old; check currently supported Aurora MySQL versions and update the `AllowedValues` and `DBEngineMap` mapping before deploying.

## Security notes

- `DBPermitLocation` defaults to `0.0.0.0/0`. Instances are not publicly accessible, but anything in (or routed into) the VPC can reach 3306; set a narrow CIDR or switch to a source security group.
- The secret uses the default AWS-managed KMS key (custom `KmsKeyId` is commented out) and has no rotation configured.
- The secret has `DeletionPolicy: Delete`, while the cluster is snapshotted on delete: the final snapshot will outlive the credentials secret.
- No IAM database authentication, enhanced monitoring or log exports are configured.

## Cleanup

```bash
# If deletion protection is enabled (default), disable it first
aws rds modify-db-cluster --db-cluster-identifier <CLUSTER_ID> --no-deletion-protection --apply-immediately
aws cloudformation delete-stack --stack-name <STACK_NAME>
```

The cluster has `DeletionPolicy: Snapshot`, so a final DB cluster snapshot is retained; delete it manually with `aws rds delete-db-cluster-snapshot` when no longer needed.

## License

MIT — see [LICENSE](LICENSE).
