# Run Ullo on AWS

The included hosting template deploys the applications on EC2 using an existing
S3 bucket and RDS PostgreSQL database. It does not provision the entire data
platform from an empty account. Use the local walkthrough in the
[README](../README.md#quick-start) for the fastest evaluation.

## Prerequisites

- Your own AWS profile and permission to create the hosting resources.
- An S3 bucket in the selected region, with demo CSVs under `bronze/`.
- An RDS PostgreSQL instance with database `dataops`, an RDS-managed master
  credential in Secrets Manager, and the included
  [incident schema](../db/init/001_create_dataops_incidents.sql) applied.
- A public-IP-enabled subnet in the RDS VPC with working internet access.
  The helper selects a matching subnet from that VPC.
- Access to the Bedrock report model in the selected region. The hosting
  configuration uses `amazon.nova-micro-v1:0`.

All commands below use example names. Replace them with resources you own.
Deployment provisions billable resources.

## Use AWS from your local applications

1. Copy `.env.aws.example` to `.env.aws.local`.
2. Supply your database URL, S3 bucket, region, and AWS profile. URL-encode
   special characters in database credentials. Leave the emulator endpoint,
   dummy access key, and dummy secret key empty.
3. Ensure your computer has a network route and authorized database access.
4. Start the applications:

```bash
python scripts/run.py dagster --env-file .env.aws.local
python scripts/run.py dashboard --env-file .env.aws.local
```

Use separate terminals. This mode executes Python on your computer while data
and incidents reside in AWS. It requires a reachable database; the hosted mode
below runs the worker in the RDS VPC.

## Deploy the hosted applications

Review [`deploy/aws/template.yaml`](../deploy/aws/template.yaml). It adds an EC2
server, encrypted root disk, Elastic IP, server role, generated web-login secret,
web security group, and a database ingress rule scoped to the server group.

Optional template checks:

```bash
python -m pip install cfn-lint
cfn-lint deploy/aws/template.yaml --regions us-east-1
```

Package the source locally, then create a new hosting stack:

```bash
python scripts/deploy_team.py package
python scripts/deploy_team.py deploy --profile default --region us-east-1 --stack ullo-demo --bucket YOUR_BUCKET --database YOUR_RDS_INSTANCE
```

The helper validates the template, creates a change set, checks that its changes
only add resources, and executes it. Use a new stack name for initial deployment.
The `--database` argument is the RDS instance identifier, not its hostname.

Before the first data-quality run, upload the sample files:

```bash
aws s3 cp sample_data/clean_orders.csv s3://YOUR_BUCKET/bronze/orders.csv --profile default --region us-east-1
aws s3 cp sample_data/high_nulls.csv s3://YOUR_BUCKET/bronze/high_nulls.csv --profile default --region us-east-1
aws s3 cp sample_data/unexpected_column.csv s3://YOUR_BUCKET/bronze/unexpected_column.csv --profile default --region us-east-1
```

The source package deliberately excludes datasets; upload them separately.
The default server is `t3a.large`. Installation downloads dependencies and model
weights, so a completed CloudFormation stack does not mean the app is ready yet.

## Open the demo

```bash
python scripts/deploy_team.py status --profile default --region us-east-1 --stack ullo-demo
python scripts/deploy_team.py login --profile default --region us-east-1 --stack ullo-demo
```

`status` prints the generated application URLs. `login` retrieves the generated
credentials into `.deployment/team-access.txt`. That directory is excluded from
Git and deployment archives. Share the login privately.

Caddy serves HTTPS through an IP-derived `sslip.io` hostname and protects both
applications with a shared login. EC2 uses its instance role to access S3,
Secrets Manager, and Bedrock. No local `.env` files are uploaded.

The server starts these services:

| Service | Purpose |
| --- | --- |
| `dataops-web` | Dagster UI |
| `dataops-daemon` | Sensors and queued run launcher |
| `dataops-dashboard` | Streamlit incident review |
| `caddy` | HTTPS and authentication |

Hosted jobs run one at a time. The incident sensor starts stopped, and SNS
notifications are disabled in the hosting configuration. Hosted outputs use
`silver/shared-demo/` and `quarantine/shared-demo/`.

## Update and operate

Publish an updated source archive to an existing stack:

```bash
python scripts/deploy_team.py update --profile default --region us-east-1 --stack ullo-demo --bucket YOUR_BUCKET
```

This updates application source through Systems Manager. Changes to the hosting
template require a separate infrastructure update.

Use Systems Manager Session Manager to investigate bootstrap or service errors;
the template does not expose SSH. On the instance:

```bash
sudo systemctl status dataops-web dataops-daemon dataops-dashboard caddy
sudo journalctl -u dataops-daemon -n 100 --no-pager
sudo tail -n 100 /var/log/cloud-init-output.log
```

Stop or start the demo server with:

```bash
python scripts/deploy_team.py stop --profile default --region us-east-1 --stack ullo-demo
python scripts/deploy_team.py start --profile default --region us-east-1 --stack ullo-demo
```

Stopping EC2 does not stop RDS or remove stored objects, the root volume, or the
reserved address. Their applicable charges continue. When finished, remove the
hosting stack and any separately provisioned resources you no longer need using
your normal AWS cleanup process.
