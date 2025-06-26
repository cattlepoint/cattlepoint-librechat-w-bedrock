# DEMO ONLY - Requires immediate hardening to prevent anonymous internet users from using
### This project is vibe-coded; use at your own risk

# cattlepoint‑librechat‑w‑bedrock

Deploy a demo of [LibreChat](https://github.com/danny-avila/LibreChat) instance on AWS that speaks directly to Amazon Bedrock models. The solution is expressed in a single CloudFormation template plus an optional CodePipeline for Git‑ops style continuous delivery.

## Key Features

* ⚡ **Graviton Spot EC2** (`t4g.small`) keeps runtime cost <\$10/mo.
* 🛡️ Traffic terminates at a regional **Application Load Balancer** and is fronted by **CloudFront** (TLS 1.2, HTTP/2).
* 🔐 The instance role is granted **Amazon Bedrock Full Access** only; data is stored in a private **S3** bucket and an optional **DynamoDB** table (pay‑per‑request).
* 🔄 **Custom Lambda resources** automatically generate and upload a minimal `.env` file and `librechat.yaml` on first deploy.
* 🛠️ **One‑click rollback**—all resources (including data) are cleaned up by the same stack when it is deleted.
* ♻️ **CI/CD pipeline** template pushes changes from GitHub → CodePipeline → CloudFormation in under two minutes.

## Prerequisites

| Tool                | Minimum version            | Notes                                                                                            |
| ------------------- | -------------------------- | ------------------------------------------------------------------------------------------------ |
| AWS CLI             |  2.16                      | Configure with credentials that can create IAM, S3, CloudFront, EC2, and CodePipeline resources. |
| Python              |  3.11                      | Only needed locally for linting.                                                                 |
| Region              | us‑east‑1/‑2, us‑west‑1/‑2 | Amazon Bedrock Graviton support varies; pick one of the listed regions.                          |
| GitHub <sup>✱</sup> | n/a                        | Required only when you use the pipeline template.                                                |

<sup>✱</sup> A [CodeStar Connections](https://docs.aws.amazon.com/dtconsole/latest/userguide/connections-create-github.html) ARN is needed so CodePipeline can pull from your repo.

## Deployment

### Option A – Manual (single command)

```bash
aws cloudformation deploy \
  --stack-name librechat \
  --template-file cattlepoint-librechat-w-bedrock.yaml \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

When the stack status reaches **CREATE\_COMPLETE** the app is available at the `ChatWebsiteURL` output.

### Option B – Automated (recommended)

```bash
aws cloudformation create-stack \
  --stack-name librechat-pipeline \
  --template-body file://cattlepoint-librechat-w-bedrock-pipeline.yaml \
  --parameters ParameterKey=ConnectionArn,ParameterValue=<your-connection-arn> \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

Every commit to the default branch will redeploy your LibreChat stack.

### Tearing down

```bash
aws cloudformation delete-stack --stack-name librechat-pipeline   # if used
aws cloudformation delete-stack --stack-name librechat
```

All application data in S3 and DynamoDB is erased automatically by the built‑in cleanup Lambda.

## Outputs

| Name                 | Description                                |
| -------------------- | ------------------------------------------ |
| `ChatWebsiteURL`     | CloudFront HTTPS endpoint for LibreChat    |
| `ConfigS3BucketName` | Bucket storing `.env` and `librechat.yaml` |
| `DynamoDBTableName`  | Chat history table name                    |

## Linting

```bash
pip install -r requirements-linters.txt
yamllint .
cfn-lint **/*.yaml
ruff .
black --check .
isort --check-only .
mypy .
bandit -r .
```

## Cost Estimate (us‑east‑1)

| Component                                       | Monthly USD |
| ----------------------------------------------- | ----------- |
| t4g.small Spot (100% utilisation)               | \~\$5.40    |
| S3 (1 GB) + PUT/GET                             | <\$0.10     |
| DynamoDB pay‑per‑request (≈1 K RCU/WCU per day) | <\$1        |
| CloudFront (100 GB data transfer out)           | \~\$8.20    |
| **Total**                                       | **≈ \$15**  |

## Customisation

The `.env` and `librechat.yaml` files uploaded to the S3 bucket take precedence over the defaults inside the AMI. Edit them and restart the EC2 instance to apply changes.

## License

MIT – see [`LICENSE`](LICENSE)