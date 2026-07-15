# Week 6 Day 3 — AWS Substrate (CloudFormation)

## What was built today

Four CloudFormation stacks provisioning the AWS substrate for the expense-project capstone, deployed via the create-change-set → describe-change-set → execute-change-set review-first flow.

## Stacks

| Stack | File | Description |
|---|---|---|
| `expense-bootstrap-dev` | `cfn/expense-bootstrap-dev.yaml` | Hardened S3 bucket + IAM role `expense-api-cfn-deploy` for GitHub Actions OIDC |
| `expense-network-dev` | `cfn/expense-network-dev.yaml` | 3-AZ VPC, public/private subnets, IGW, NAT GW (1 in dev, 3 in HA), app security group |
| `expense-artifacts-dev` | `cfn/expense-artifacts-dev.yaml` | Hardened S3 artefact bucket with KMS, PAB, lifecycle, deny-non-TLS |
| `expense-app-dev` | `cfn/expense-app-dev.yaml` | RDS Postgres 16.3 + Secrets Manager credentials via dynamic reference |

## Key design decisions

- **No `Action: '*'` or `Resource: '*'`** — every IAM policy enumerates actions explicitly and scopes to `stack/expense-*` ARNs
- **Secrets Manager dynamic reference** — DB master password uses `{{resolve:secretsmanager:expense/dev/db-master:SecretString:password}}`, never a `NoEcho` Parameter (which leaks via `DescribeStacks`)
- **`DeletionPolicy: Retain` + `UpdateReplacePolicy: Retain`** on every stateful resource — protects against both stack delete and property-change replacement
- **Conditions gate NAT cost** — `IsDev` deploys 1 NAT GW; `IsHA` (staging/prod) deploys 3 — one per AZ
- **`!Cidr [!Ref VpcCidr, 6, 8]`** — one VpcCidr parameter drives all 6 subnet CIDRs automatically

## CI added

`.github/workflows/cfn-validate.yml` — runs cfn-lint + cfn-nag + `aws cloudformation validate-template` on every PR touching `cfn/`. Uses OIDC to assume `expense-api-cfn-deploy`. All third-party actions pinned by SHA.

## Cross-stack references

The app stack consumes the network stack outputs via `!ImportValue`:
- `expense-network-dev-PrivateSubnets` → split with `!Split` for RDS subnet group
- `expense-network-dev-VpcId` → DB security group VPC
- `expense-network-dev-AppSgId` → DB security group ingress source

## Deploy order

1. `expense-bootstrap-dev`
2. `expense-artifacts-dev`
3. `expense-network-dev`
4. `expense-app-dev`

## Related

- Full infra docs: [`expense-api/INFRA.md`](expense-api/INFRA.md)
- CI pipeline: [`.github/workflows/cfn-validate.yml`](.github/workflows/cfn-validate.yml)
- Application repo: [Syed-web-coder/expense-project](https://github.com/Syed-web-coder/expense-project)
