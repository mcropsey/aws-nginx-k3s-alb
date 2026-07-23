# AWS K3s Cluster Deployment Runbook (fixed template)

Uses `aws-k3s-cluster-fixed.yaml`, which corrects the following bugs in the original template:

- Added an Internet Gateway + public route table (original had no route to the internet — instances would get no public IP and be unreachable).
- Fixed `SourceSecurityGroupName` → `SourceSecurityGroupId` (the original value doesn't work outside EC2-Classic and would have failed stack creation).
- Wired up the unused `AdminIPRange` parameter to actually restrict SSH (port 22), on both the control plane and the nginx box.
- Added the missing `k3s` install step (the original never installed k3s despite the template's name) and wrote a kubeconfig to `~/.kube/config` for the `mcropsey` user.
- Replaced the `<ALB-DNS>` placeholder in nginx's `proxy_pass` with the control plane's real private IP (there was no ALB defined anywhere in the template).
- Added explicit SG rules so nginx can reach the control plane's ingress on 80/443, and added `ControlPlanePublicIP` / `ControlPlanePrivateIP` outputs.
- Renamed all resource logical IDs to remove hyphens (e.g. `mcropsey-lab-VPC` → `mcropseylabVPC`). CloudFormation logical IDs must be alphanumeric only — the original template would fail immediately with `Template format error: Resource name ... is non alphanumeric.`
- Added the required `GroupDescription` property to both security groups (`mcropseylabClusterSG`, `mcropseylabNginxSG`). AWS rejects `AWS::EC2::SecurityGroup` resources without it — the original template would pass template validation but fail at creation time with `The request must contain the parameter GroupDescription`.
- Replaced the `cfn-init`/`AWS::CloudFormation::Init` mechanism with a plain bash UserData script on both instances. The original relied on `yum install -y aws-cfn-bootstrap`, but that package isn't available in AlmaLinux's default repos (it's an Amazon Linux thing) — the install fails, and because the script uses `bash -xe`, everything after it (creating `mcropsey`, copying SSH keys, disabling firewalld/SELinux, installing k3s) never runs. The instance boots but only has the default `ec2-user` login; `mcropsey`/`almalinux` SSH attempts get `Permission denied`. The new script does the same work directly, no extra package needed.
- Added `setenforce 0` alongside the `sed` that edits `/etc/selinux/config`. Editing the config file only changes SELinux's mode on the *next reboot* — the currently running kernel stays in whatever mode it booted with (Enforcing, by default). Without `setenforce 0`, SELinux silently blocks nginx's proxy connection to the control plane with `connect() ... failed (13: Permission denied)`, which surfaces as a `502 Bad Gateway` from nginx with no obvious cause unless you check `/var/log/nginx/error.log`.

---

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
You should see your account ID and user ARN.

---

## Step 1 — Create an EC2 Key Pair

One-time only. Saved to `~/.ssh` (not `~/Downloads` — the conventional location for keys):

```bash
mkdir -p ~/.ssh
aws ec2 create-key-pair \
  --key-name mcropsey-lab-key \
  --region us-east-2 \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/mcropsey-lab-key.pem && chmod 400 ~/.ssh/mcropsey-lab-key.pem
```

Verify the key pair is registered in AWS:
```bash
aws ec2 describe-key-pairs --region us-east-2 --query 'KeyPairs[*].KeyPairName' --output table
```

**Gotcha:** if `create-key-pair` fails with `InvalidKeyPair.Duplicate`, the key pair already exists in AWS from an earlier attempt. Check whether you actually have a valid local copy of the private key — AWS never lets you re-download it:
```bash
ls -la ~/.ssh/mcropsey-lab-key.pem
head -1 ~/.ssh/mcropsey-lab-key.pem
```
If the file is missing, empty (0 bytes), or doesn't start with `-----BEGIN RSA PRIVATE KEY-----` (this can happen if a prior command redirected output before erroring out), the key is unrecoverable. Delete and recreate it:
```bash
aws ec2 delete-key-pair --key-name mcropsey-lab-key --region us-east-2
aws ec2 create-key-pair \
  --key-name mcropsey-lab-key \
  --region us-east-2 \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/mcropsey-lab-key.pem && chmod 400 ~/.ssh/mcropsey-lab-key.pem
```

---

## Step 2 — Confirm the AMI ID

| Field | Value |
|-------|-------|
| AMI ID | `ami-053975fce80e8ee1a` |
| Name | AlmaLinux OS 9.8.20260526 x86\_64 |
| Region | us-east-2 |
| Channel | Community AMI (free, no subscription required) |

**Gotcha:** the original runbook used `ami-02807aa666b76fff5`, an **AWS Marketplace** AMI. Marketplace AMIs require accepting the product's terms/subscribing in the AWS console first — the AWS CLI/CloudFormation can't do this for you, and launching one without subscribing fails with:
```
Resource handler returned message: "In order to use this AWS Marketplace product you need to
accept terms and subscribe. To do so please visit https://aws.amazon.com/marketplace/pp?sku=..."
(Service: Ec2, Status Code: 401 ... HandlerErrorCode: AccessDenied)
```
This triggers a full stack rollback since it happens mid-creation (see "If the stack rolls back or fails" below).

AlmaLinux publishes AMIs through two channels: **Marketplace** (needs subscription) and **Community** (free, no subscription — just a public AMI ID). We use the Community channel now. AlmaLinux maintains a full, current list of Community AMI IDs per region/arch here: https://wiki.almalinux.org/cloud/AWS.html (also available as CSV). Re-check that page if this AMI ID ages out — AlmaLinux publishes new point releases periodically and old AMI IDs can be deregistered.

---

## Step 3 — Confirm Your Public IP

The original runbook had `167.237.109.9` on file — confirm it's still current before deploying, since this becomes the only IP allowed to SSH in:
```bash
curl -s ifconfig.me
```

---

## Step 4 — Deploy the CloudFormation Stack

Make sure `aws-k3s-cluster-fixed.yaml` is saved locally, e.g. at `~/Downloads/aws-k3s-cluster-fixed.yaml`.

```bash
aws cloudformation create-stack \
  --stack-name mcropsey-lab-k3s-cluster \
  --template-body file://$HOME/Downloads/aws-k3s-cluster-fixed.yaml \
  --parameters \
    ParameterKey=AlmaLinux9AMIId,ParameterValue=ami-053975fce80e8ee1a \
    ParameterKey=KeyPairName,ParameterValue=mcropsey-lab-key \
    ParameterKey=AdminIPRange,ParameterValue=<YOUR-IP>/32 \
  --region us-east-2
```
Replace `<YOUR-IP>` with the output from Step 3.

On success:
```json
{
    "StackId": "arn:aws:cloudformation:us-east-2:XXXXXXXXXXXX:stack/mcropsey-lab-k3s-cluster/..."
}
```

---

## Step 5 — Monitor Stack Creation

```bash
aws cloudformation describe-stacks \
  --stack-name mcropsey-lab-k3s-cluster \
  --region us-east-2 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

| Status | Meaning |
|--------|---------|
| `CREATE_IN_PROGRESS` | Still building — wait and re-check |
| `CREATE_COMPLETE` | Done — proceed to Step 6 |
| `ROLLBACK_IN_PROGRESS` | Something failed — check events below |
| `ROLLBACK_COMPLETE` | Failed and rolled back — check events |

If it fails:
```bash
aws cloudformation describe-stack-events \
  --stack-name mcropsey-lab-k3s-cluster \
  --region us-east-2 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

This typically takes 5–8 minutes (k3s install adds time vs. the original template).

### If the stack rolls back or fails

A `CREATE_FAILED` on any resource triggers an automatic rollback (`ROLLBACK_IN_PROGRESS`), which tears down everything CloudFormation already created for this attempt. You cannot fix or retry a stack in this state — CloudFormation stacks that fail their *first* create always land in `ROLLBACK_COMPLETE`, which only allows deletion, not updates.

1. **Wait for rollback to finish.** Poll status until it reads `ROLLBACK_COMPLETE`:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name mcropsey-lab-k3s-cluster \
     --region us-east-2 \
     --query 'Stacks[0].StackStatus' \
     --output text
   ```

2. **Read the failure reason** (from the `describe-stack-events` command above) — this tells you exactly which resource failed and why.

3. **Delete the failed stack:**
   ```bash
   aws cloudformation delete-stack \
     --stack-name mcropsey-lab-k3s-cluster \
     --region us-east-2
   ```

4. **Confirm it's gone** — poll until `describe-stacks` errors with "does not exist":
   ```bash
   aws cloudformation describe-stacks \
     --stack-name mcropsey-lab-k3s-cluster \
     --region us-east-2 \
     --query 'Stacks[0].StackStatus' \
     --output text
   ```

5. **Fix the underlying issue** (template bug, bad parameter value, IAM permissions, etc.), then re-run the `create-stack` command from Step 4.

Real failures hit while building this template, all now fixed in `aws-k3s-cluster-fixed.yaml`:
- Non-alphanumeric logical resource IDs (rejected at template-parse time, before any resources are touched — no rollback needed, just fix and resubmit).
- Missing `GroupDescription` on the security groups (rejected mid-creation by the EC2 API — this one *does* trigger a rollback of whatever was already created, hence the delete-and-retry cycle above).
- Wrong AMI channel — the original AMI ID was a Marketplace product requiring manual subscription (see Step 2).
- `aws-cfn-bootstrap` not available on AlmaLinux, silently breaking every step after it in UserData (see the bullet in the intro). **Important:** this one won't show up as `CREATE_FAILED` in `describe-stack-events` — the EC2 instance and stack both report success, because CloudFormation only tracks whether the *instance itself* launched, not whether UserData succeeded inside it. If SSH access doesn't work after a clean `CREATE_COMPLETE`, check the instance's boot log:
  ```bash
  aws ec2 get-console-output --instance-id <INSTANCE-ID> --region us-east-2 --output text | \
    grep -iE "cfn-init|cloud-init|useradd|authorized_keys|ec2-user|almalinux|error|failed"
  ```
  The default login for this AMI is `ec2-user` (cloud-init injects your key pair there automatically, independent of anything in UserData) — useful for debugging when the `mcropsey` user never got created.

---

## Step 6 — Get the Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name mcropsey-lab-k3s-cluster \
  --region us-east-2 \
  --query 'Stacks[0].Outputs' \
  --output table
```

You'll get three values now instead of one:
- `NginxPublicIP` — public entry point (ports 80/443)
- `ControlPlanePublicIP` — for SSH / kubectl access
- `ControlPlanePrivateIP` — internal IP nginx proxies to (informational only)

---

## Step 7 — SSH into the Instances

Control plane (k3s + kubeconfig already set up for the `mcropsey` user):
```bash
ssh -i ~/.ssh/mcropsey-lab-key.pem mcropsey@<CONTROL-PLANE-PUBLIC-IP>
```
Once in, verify the cluster:
```bash
kubectl get nodes
```

Nginx ingress:
```bash
ssh -i ~/.ssh/mcropsey-lab-key.pem mcropsey@<NGINX-PUBLIC-IP>
```

Both instances now also allow SSH directly from your admin IP (fixed — the original only allowed it in one direction and never for the control plane).

Alternative way to find IPs:
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=mcropsey-lab-K3s-Cluster" \
  --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table \
  --region us-east-2
```

---

## Step 8 — Verify End-to-End

From your **local machine** (not over SSH), hit the nginx public IP on port 80:
```bash
curl -v http://<NGINX-PUBLIC-IP>/
```
A `404 page not found` response with `Server: nginx` in the headers is actually success — it means nginx proxied the request through to the control plane's built-in Traefik ingress controller, and Traefik correctly reported that no ingress route is configured yet (nothing's deployed to the cluster). A `502 Bad Gateway` instead means nginx couldn't reach the control plane — check `/var/log/nginx/error.log` on the nginx box. A `connect() ... failed (13: Permission denied)` in that log means SELinux is still in Enforcing mode; run `sudo setenforce 0` on the nginx instance for an immediate fix (this shouldn't happen with the current template, which sets this automatically, but can occur if you're debugging an older deployment).

Note: only port 80 has an active listener on the nginx box — the security group opens 443 too, but the config only defines a `listen 80` server block. Fine for this lab; add a `listen 443 ssl` block with a cert if you need HTTPS on the nginx entry point.

---

## Teardown (when done)

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

SSH key authentication only — no password set on the `mcropsey` user. Access is granted by copying the EC2 key pair's `authorized_keys` from `ec2-user` to `mcropsey`. SSH (port 22) is now restricted to `AdminIPRange` on both instances; the k3s API (6443) is also restricted to `AdminIPRange` rather than open to `0.0.0.0/0` as in the original.
