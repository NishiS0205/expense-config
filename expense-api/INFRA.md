# expense-api/INFRA.md
# How this service's AWS substrate is provisioned and what
# deviated from the cfn-author Skill's defaults.

## Stack layout

| Stack | What it holds |
|---|---|
| `expense-bootstrap-dev` | Hardened S3 artefact bucket (KMS, PAB×4, versioning, lifecycle, deny-non-TLS, Retain×2) + IAM role `expense-api-cfn-deploy` for GitHub Actions OIDC. Deploy first. |
| `expense-network-dev` | 3-AZ VPC, 6 subnets (public + private), IGW, 1 NAT GW in dev / 3 in HA (Conditions-gated by EnvName), application security group (ingress 8080 from VPC CIDR only, egress 443 only). Exports VpcId, VpcCidr, PrivateSubnets, PublicSubnets, AppSecurityGroupId. |
| `expense-artifacts-dev` | Hardened S3 artefact bucket (KMS, PAB×4, versioning, 90-day STANDARD_IA transition, deny-non-TLS + deny-unencrypted-put, Retain×2). Independent of the network stack. |
| `expense-app-dev` | RDS Postgres 16.3 + DB SG + DB subnet group + Secrets Manager master credentials. Consumes the network stack via `!ImportValue`. DB master password resolved via `{{resolve:secretsmanager:expense/dev/db-master:SecretString:password}}` — NEVER a NoEcho Parameter. |

## Deploy order

1. `expense-bootstrap-dev` — bootstrap bucket + CFN-deploy IAM role.
2. `expense-artifacts-dev` — artefact bucket (no cross-stack dependency).
3. `expense-network-dev` — VPC + subnets + SG.
4. `expense-app-dev` — RDS + secret + SecretTargetAttachment (requires network stack CREATE_COMPLETE).

## ChangeSet flow (every stack, every change)

```bash
aws cloudformation create-change-set \
  --stack-name <stack> \
  --change-set-name <name> \
  --change-set-type CREATE_OR_UPDATE \
  --template-body file://cfn/<template>.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=EnvName,ParameterValue=dev \
  --region us-east-1

aws cloudformation describe-change-set \
  --stack-name <stack> --change-set-name <name> \
  --region us-east-1
# paste the JSON diff into the PR body

aws cloudformation execute-change-set \
  --stack-name <stack> --change-set-name <name> \
  --region us-east-1
```

The describe-change-set output enumerates every Replace, Modify, Add, and Remove. PR reviewers read the diff before the execute step.

## Drift detection (deliberate verification — Task 4)

```bash
aws cloudformation detect-stack-drift \
  --stack-name expense-network-dev \
  --region us-east-1
# capture the StackDriftDetectionId, poll:
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id> \
  --region us-east-1
# final per-resource report:
aws cloudformation describe-stack-resource-drifts \
  --stack-name expense-network-dev \
  --region us-east-1 \
  --query "StackResourceDrifts[?StackResourceDriftStatus!='IN_SYNC']"
```

For Task 4 we deliberately drifted the network stack by console-editing the app SG (added a stray egress rule), confirmed the drift report flagged `ExpenseAppSecurityGroup` as `MODIFIED`, then reverted by deleting the stray rule and re-running drift detection until `IN_SYNC`.

## cfn-author Skill audit notes

**Suggestion accepted:** The Skill correctly paired `DeletionPolicy: Retain` on the RDS instance, but omitted `UpdateReplacePolicy: Retain`. We added it to all stateful resources (`DbMasterSecret`, `DbInstance`). Without `UpdateReplacePolicy: Retain`, a property change that triggers resource replacement (e.g. renaming the DB identifier) would delete the old instance on the next apply — identical blast radius to a stack delete. Both policies are required together (Section 9 sticking point #1).

**Suggestion rejected:** The Skill generated a `NoEcho: true` Parameter for the DB master password. We rejected this and used a Secrets Manager `GenerateSecretString` + `{{resolve:secretsmanager:...}}` dynamic reference instead. The reason: a `NoEcho` Parameter value is still readable via `cloudformation:DescribeStacks` by anyone with that IAM permission — it only hides the value in the console UI. The dynamic reference means CFN never holds the password value at all; only the secret ARN is stored in stack state (Section 9 sticking point #2).
