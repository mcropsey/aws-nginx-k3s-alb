# AWS K3s Cluster Deployment Runbook

## Prerequisites

### 1. Install AWS CLI (if not already installed)
```bash
brew install awscli
```
Verify:
```bash
aws --version
```

### 2. Configure AWS credentials
```bash
aws configure
```
You'll be prompted for:
- **AWS Access Key ID** — from your IAM user in the AWS console
- **AWS Secret Access Key** — from your IAM user in the AWS console
- **Default region** — enter `us-east-2`
- **Default output format** — enter `json`

Verify credentials work:
```bash
aws sts get-caller-identity
```
You should see your account ID and user ARN. If you get an error, your credentials are wrong or expired.

---

## Step 1 — Create an EC2 Key Pair

You need this to SSH into the instances after they launch. You only need to do this once.

```bash
aws ec2 create-key-pair \
  --key-name mcropsey-lab-key \
  --region us-east-2 \
  --query 'KeyMaterial' \
  --output text > ~/Downloads/mcropsey-lab-key.pem && chmod 400 ~/Downloads/mcropsey-lab-key.pem
```

Verify it was created:
```bash
aws ec2 describe-key-pairs --region us-east-2 --query 'KeyPairs[*].KeyPairName' --output table
```
You should see `mcropsey-lab-key` in the output.

---

## Step 2 — Confirm the AMI ID

The AlmaLinux 9 AMI for `us-east-2` has already been looked up:

| Field | Value |
|-------|-------|
| AMI ID | `ami-02807aa666b76fff5` |
| Name | AlmaLinux 9.8 x86\_64 |
| Region | us-east-2 |

If you want to re-verify or get a fresher AMI:
```bash
aws ec2 describe-images \
  --owners aws-marketplace \
  --filters "Name=name,Values=*AlmaLinux*9*" \
  --query 'sort_by(Images,&CreationDate)[-1].[ImageId,Name]' \
  --output table \
  --region us-east-2
```

---

## Step 3 — Confirm Your Public IP

Your current public IP (for SSH access restriction):

```
167.237.109.9
```

To re-check if your IP has changed (e.g. you're on a different network):
```bash
curl -s ifconfig.me
```

---

## Step 4 — Deploy the CloudFormation Stack

Run this from any directory. The template path is absolute so location doesn't matter.

```bash
aws cloudformation create-stack \
  --stack-name mcropsey-lab-k3s-cluster \
  --template-body file:///Users/mcropsey/Downloads/aws-k3s-cluster.yaml \
  --parameters \
    ParameterKey=AlmaLinux9AMIId,ParameterValue=ami-02807aa666b76fff5 \
    ParameterKey=KeyPairName,ParameterValue=mcropsey-lab-key \
    ParameterKey=AdminIPRange,ParameterValue=167.237.109.9/32 \
  --region us-east-2
```

On success you'll see output like:
```json
{
    "StackId": "arn:aws:cloudformation:us-east-2:XXXXXXXXXXXX:stack/mcropsey-lab-k3s-cluster/..."
}
```

---

## Step 5 — Monitor Stack Creation

Check status (run repeatedly until `CREATE_COMPLETE`):
```bash
aws cloudformation describe-stacks \
  --stack-name mcropsey-lab-k3s-cluster \
  --region us-east-2 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Possible statuses:
| Status | Meaning |
|--------|---------|
| `CREATE_IN_PROGRESS` | Still building — wait and re-check |
| `CREATE_COMPLETE` | Done — proceed to Step 6 |
| `ROLLBACK_IN_PROGRESS` | Something failed — check events below |
| `ROLLBACK_COMPLETE` | Failed and rolled back — check events |

If it fails, see what went wrong:
```bash
aws cloudformation describe-stack-events \
  --stack-name mcropsey-lab-k3s-cluster \
  --region us-east-2 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

---

## Step 6 — Get the Nginx Public IP

Once the stack is `CREATE_COMPLETE`, get the Nginx ingress IP:
```bash
aws cloudformation describe-stacks \
  --stack-name mcropsey-lab-k3s-cluster \
  --region us-east-2 \
  --query 'Stacks[0].Outputs[?OutputKey==`NginxPublicIP`].OutputValue' \
  --output text
```

---

## Step 7 — SSH into the Instances

Control plane:
```bash
ssh -i ~/Downloads/mcropsey-lab-key.pem almalinux@<CONTROL-PLANE-IP>
```

Nginx ingress:
```bash
ssh -i ~/Downloads/mcropsey-lab-key.pem almalinux@<NGINX-PUBLIC-IP>
```

> Replace `<CONTROL-PLANE-IP>` with the instance IP from the EC2 console or:
> ```bash
> aws ec2 describe-instances \
>   --filters "Name=tag:Project,Values=mcropsey-lab-K3s-Cluster" \
>   --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
>   --output table \
>   --region us-east-2
> ```

---

## Teardown (when done)

To delete all resources and stop incurring charges:
```bash
aws cloudformation delete-stack \
  --stack-name mcropsey-lab-k3s-cluster \
  --region us-east-2
```

Monitor deletion:
```bash
aws cloudformation describe-stacks \
  --stack-name mcropsey-lab-k3s-cluster \
  --region us-east-2 \
  --query 'Stacks[0].StackStatus' \
  --output text
```
Stack is gone when this returns an error saying the stack does not exist.

---

## Security Note

The CloudFormation template uses SSH key authentication only — no password is set on the `mcropsey` user. Access is granted by copying the EC2 key pair's `authorized_keys` from the `ec2-user` to `mcropsey`. Password-based SSH login is not configured.
