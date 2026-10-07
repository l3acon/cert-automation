# Certificate Automation

Automated Let's Encrypt certificate issuance and distribution using **Ansible Automation Platform (AAP)**, **AWS Route53**, and **AWS Secrets Manager**.

This project uses AAP Config as Code (`infra.aap_configuration.dispatch`) to declaratively build all AAP resources — credentials, projects, inventories, job templates, and workflows — from a single deploy playbook.

## How It Works

```
         ┌──────────────┐
         │  AAP Survey   │  User provides: Ansible hostname, Route53 zone, ACME env
         └──────┬───────┘
                │
    ┌───────────▼───────────┐
    │   Issue Certificate    │  1. EC2 lookup by Name tag → resolve FQDN + IP
    │   (localhost)          │  2. Ensure A record in Route53
    │                        │  3. ACME DNS-01 challenge via Route53 TXT record
    │                        │  4. Store cert + key in AWS Secrets Manager
    └───────────┬───────────┘
                │ on success
    ┌───────────▼───────────┐
    │  Distribute Certificate│  1. Read cert from Secrets Manager
    │  (target hosts)        │  2. Write fullchain + key to target hosts
    └───────────────────────┘
```

### Credential Flow

Secrets are managed through AWS Secrets Manager, not stored directly in AAP:

```
Secret Zero (in AAP)                 AWS Secrets Manager
┌────────────────────────┐          ┌──────────────────────────────────────┐
│ AWS SM Lookup credential│─────────▶│ cert-automation/                     │
│ (aws_access_key,        │ pull at │   machine-credential/ssh_key_data    │
│  aws_secret_key)        │ launch  │   certs/<host>/<env> → cert + key    │
└────────────────────────┘         └──────────────────────────────────────┘
```

- **Machine credential** SSH key is pulled from Secrets Manager at job launch time via AAP [credential input sources](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_execution/controller-credential-plugins)
- **TLS certificates** are written to Secrets Manager after issuance and read from there during distribution — no ephemeral AAP credentials
- The **AWS SM Lookup credential** is the only "secret zero" stored directly in AAP

## Prerequisites

- **Ansible Automation Platform 2.5+** with the following pre-existing credentials:
  - `AWS` — Amazon Web Services credential (Route53 + EC2 + Secrets Manager access)
  - `AAP Credential` — Red Hat Ansible Automation Platform credential (for cross-JT orchestration)
- **AWS** account with:
  - Route53 hosted zone for your domain
  - EC2 instances tagged with a `Name` tag (e.g., `aws_rhel9`)
  - Secrets Manager access (the IAM user/role needs `secretsmanager:*` permissions)
- **Local workstation** with:
  - `ansible-core` 2.16+
  - `infra.aap_configuration` collection
  - `community.aws`, `amazon.aws`, `community.crypto` collections
  - Python packages: `boto3`, `botocore`

### Install local dependencies

```bash
pip install boto3 botocore
ansible-galaxy collection install infra.aap_configuration community.aws amazon.aws community.crypto
```

## Project Structure

```
cert-automation/
├── deploy_certs.yml                          # Run locally — deploys all CaC to AAP
│
├── playbooks/certs/                          # Run inside AAP job templates
│   ├── issue_cert.yml                        #   ACME DNS-01 issuance + SM storage
│   ├── distribute_cert.yml                   #   Read from SM, deploy to hosts
│   └── notify_failure.yml                    #   Workflow failure notification stub
│
├── roles/
│   └── cert_automation/
│       ├── defaults/main.yml                 #   Configurable settings (knobs)
│       ├── vars/main.yml                     #   CaC definitions (controller_*)
│       └── tasks/main.yml                    #   Dispatches CaC to AAP
│
├── collections/requirements.yml              # Collections needed by AAP (auto-installed)
├── execution_environments/cert_automation/   # EE build definition (optional)
├── examples/                                 # Sanitized credential templates
│   ├── env
│   └── aws.creds.yml
├── ansible.cfg
└── .gitignore
```

## Setup

### 1. Create the `.env` file

```bash
cp examples/env .env
```

Edit `.env` with your AAP and GitHub credentials:

```bash
export CONTROLLER_HOST="https://aap.example.com/"
export CONTROLLER_USERNAME="admin"
export CONTROLLER_PASSWORD="your-password"
export CONTROLLER_VERIFY_SSL="false"
export GITHUB_PAT="ghp_xxxxxxxxxxxxxxxxxxxx"
```

> The GitHub PAT is used as an SCM credential so AAP can pull this repo. If the repo is public, a PAT with no scopes is sufficient.

### 2. Provide AWS credentials

The deploy playbook resolves AWS credentials and the Route53 zone automatically from a ROSA/RHDP `rosa.creds.yml` file (expected at `../mad-hatter/rosa.creds.yml` by default).

To override, pass extra vars directly:

```bash
ansible-playbook deploy_certs.yml \
  -e aws_access_key_id=AKIA... \
  -e aws_secret_access_key=... \
  -e cert_route53_zone=example.com \
  -e cert_aws_region=us-east-2
```

### 3. Seed the machine credential SSH key

The deploy playbook will attempt to read `~/.ssh/id_rsa` and store it in Secrets Manager. If your key is elsewhere, set the `MACHINE_SSH_KEY` environment variable:

```bash
export MACHINE_SSH_KEY="$(cat ~/.ssh/my_key)"
```

Or seed it manually after deploy:

```bash
aws secretsmanager put-secret-value \
  --secret-id cert-automation/machine-credential/ssh_key_data \
  --secret-string "$(cat ~/.ssh/my_key)" \
  --region us-east-2
```

### 4. Deploy to AAP

```bash
source .env
ansible-playbook deploy_certs.yml
```

This creates all AAP resources:

| AAP Resource | Name |
|---|---|
| Credential (SM Lookup) | AWS SM Lookup - Cert Automation |
| Credential (Machine) | Machine - Cert Automation |
| Credential (SCM) | GitHub PAT - Certificate Automation |
| Project | Certificate Automation |
| Inventory | Cert Automation - Localhost |
| Job Template | Cert - Issue Certificate |
| Job Template | Cert - Distribute Certificate |
| Job Template | Cert - Notify Failure |
| Workflow | Cert - Issue and Distribute |
| Workflow | Cert - Staging to Production |

## Usage

### From the AAP UI

1. Navigate to **Resources → Templates**
2. Launch **Cert - Issue and Distribute** (or the individual JTs)
3. Fill in the survey:

| Field | Example | Description |
|---|---|---|
| Ansible Hostname | `aws_rhel9` | EC2 `Name` tag of the target instance |
| Route53 Zone | `example.com` | AWS hosted zone for DNS challenges |
| ACME Environment | `staging` | `staging` for test certs, `production` for trusted certs |
| Account Email | `you@example.com` | Let's Encrypt account registration email |
| Clean up DNS TXT record? | `false` | Keep `false` for auditability |

### From the CLI

```bash
source .env
# Launch via the AAP API
curl -sk -u "$CONTROLLER_USERNAME:$CONTROLLER_PASSWORD" \
  -X POST -H "Content-Type: application/json" \
  "${CONTROLLER_HOST}api/controller/v2/workflow_job_templates/<ID>/launch/" \
  -d '{
    "extra_vars": {
      "cert_ansible_hostname": "aws_rhel9",
      "cert_route53_zone": "example.com",
      "cert_acme_env": "staging",
      "cert_account_email": "you@example.com",
      "cert_cleanup_dns_record": "false"
    }
  }'
```

## Workflows

### Cert - Issue and Distribute

Single-environment workflow with failure notifications:

```
Issue Certificate ──success──▶ Distribute Certificate
       │                              │
     failure                        failure
       │                              │
       ▼                              ▼
  Notify Failure                 Notify Failure
```

### Cert - Staging to Production

Full staged rollout — staging must succeed before production begins:

```
Staging Issue ──▶ Staging Distribute ──▶ Production Issue ──▶ Production Distribute
     │                   │                      │                      │
   failure             failure                failure                failure
     ▼                   ▼                      ▼                      ▼
  Notify              Notify                 Notify                 Notify
```

## Secrets Manager Layout

```
cert-automation/
├── machine-credential/
│   └── ssh_key_data                  # Raw SSH private key (consumed by AAP input source)
└── certs/
    └── <fqdn>/
        ├── staging                   # JSON: {fullchain, private_key, hostname, acme_env, ...}
        └── production                # JSON: {fullchain, private_key, hostname, acme_env, ...}
```

Each cert secret is tagged with `hostname`, `acme_env`, `ansible_hostname`, and `managed_by: cert-automation` for discoverability and auditing.

## How FQDN Resolution Works

The playbook resolves a target host's FQDN from just its Ansible hostname (EC2 `Name` tag):

1. `amazon.aws.ec2_instance_info` looks up the instance by `Name` tag
2. If the instance has an `FQDN` tag, that value is used
3. Otherwise, the FQDN is constructed as `<hostname>.zone` (underscores replaced with hyphens)
4. An A record is created/updated in Route53 pointing to the instance's public IP

Example: hostname `aws_rhel9` + zone `example.com` → FQDN `aws-rhel9.example.com`

## Configuration Reference

All defaults live in `roles/cert_automation/defaults/main.yml` and can be overridden via extra vars or the deploy playbook:

| Variable | Default | Description |
|---|---|---|
| `cert_aap_org` | `Default` | AAP organization |
| `cert_aap_project_scm_url` | `https://github.com/l3acon/cert-automation.git` | SCM URL for the AAP project |
| `cert_aap_ee_name` | `Default execution environment` | Execution environment name |
| `cert_acme_env` | `staging` | ACME environment (`staging` or `production`) |
| `cert_account_email` | — | Let's Encrypt account email |
| `cert_aws_region` | `us-east-2` | AWS region |
| `cert_sm_secret_prefix` | `cert-automation` | Secrets Manager path prefix |
| `cert_route53_zone` | — | Route53 hosted zone name |
| `cert_cleanup_dns_record` | `false` | Delete DNS-01 TXT record after validation |
| `cert_dest_path` | `/etc/pki/tls/certs/fullchain.crt` | Certificate install path on targets |
| `cert_key_dest_path` | `/etc/pki/tls/private/server.key` | Private key install path on targets |
| `cert_aws_credential_name` | `AWS` | Name of the pre-existing AWS credential in AAP |
| `cert_machine_credential_name` | `Machine - Cert Automation` | Machine credential name (created by CaC) |
| `cert_aap_credential_name` | `AAP Credential` | Name of the pre-existing AAP credential |

## Building the Execution Environment

An EE definition is included if you need to build a custom image:

```bash
cd execution_environments/cert_automation
ansible-builder build -t cert-automation-ee:latest
```

The EE includes `amazon.aws`, `community.aws`, `community.crypto`, `community.general`, plus Python dependencies `boto3`, `botocore`, and `cryptography`.
