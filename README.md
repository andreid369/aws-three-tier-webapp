# WordPress on AWS — CloudFormation Deployment

This template deploys a scalable WordPress stack on AWS:

- A VPC with public and private subnets across **2 Availability Zones**
- A **MySQL RDS** instance in the private subnets
- An **Auto Scaling Group** of WordPress EC2 instances (`t2.micro`) in the public subnets, installed via EC2 User Data
- An **Application Load Balancer (ALB)** distributing traffic to the ASG (min 2 / max 3 instances)
- A **CloudWatch Dashboard** showing EC2 CPU, RDS CPU, and ALB request count

> **Note on regions:** AWS does not have a region literally called "us north." Use a real US region such as `us-east-1` (N. Virginia), `us-east-2` (Ohio), `us-west-1` (N. California), or `us-west-2` (Oregon). Everything below uses `us-east-1` as an example — swap in whichever US region you prefer.

---

## Files

- `wordpress-infra.yaml` — the CloudFormation template
- `docs/architecture.png` — architecture diagram shown in this README
- `docs/architecture.drawio` — editable source of the diagram (draw.io)
- `README.md` — this guide

---

## Prerequisites

1. An AWS account with permissions to create VPCs, EC2, RDS, ELB, Auto Scaling, IAM roles, and CloudWatch dashboards.
2. The [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (`aws configure`), **or** access to the CloudFormation console.
3. An existing EC2 **Key Pair** in the target region (for SSH access). Create one if you don't have one:
   ```bash
   aws ec2 create-key-pair --key-name wordpress-key \
     --query 'KeyMaterial' --output text > wordpress-key.pem
   chmod 400 wordpress-key.pem
   ```

---

## Step-by-step: Deploying with the AWS CLI

### 1. Pick your region and Availability Zones

List the AZs available in your chosen region:

```bash
aws ec2 describe-availability-zones --region us-east-1
```

You'll use two of them (e.g. `us-east-1a` and `us-east-1b`) as parameters below.

### 2. Validate the template (optional but recommended)

```bash
aws cloudformation validate-template \
  --template-body file://wordpress-infra.yaml \
  --region us-east-1
```

### 3. Deploy the stack

```bash
aws cloudformation create-stack \
  --stack-name wordpress-stack \
  --template-body file://wordpress-infra.yaml \
  --region us-east-1 \
  --capabilities CAPABILITY_IAM \
  --parameters \
    ParameterKey=AvailabilityZone1,ParameterValue=us-east-1a \
    ParameterKey=AvailabilityZone2,ParameterValue=us-east-1b \
    ParameterKey=KeyPairName,ParameterValue=wordpress-key \
    ParameterKey=MyIPForSSH,ParameterValue=YOUR_IP/32 \
    ParameterKey=DBUser,ParameterValue=wpadmin \
    ParameterKey=DBPassword,ParameterValue='ChangeMe123!'
```

Replace:
- `YOUR_IP/32` with your actual public IP (find it with `curl ifconfig.me`), so only you can SSH in. Leaving the default `0.0.0.0/0` opens SSH to the whole internet.
- `ChangeMe123!` with a strong password (8+ characters). This becomes the RDS master password — keep it secret.

All other parameters (subnet CIDRs, instance sizes, ASG min/max, DB name, etc.) have sensible defaults and can be overridden the same way if needed — see the `Parameters` section of the template for the full list.

### 4. Watch the stack create

```bash
aws cloudformation wait stack-create-complete \
  --stack-name wordpress-stack --region us-east-1
```

This typically takes **10–15 minutes** (RDS provisioning is the slowest part). You can also watch progress live in the **CloudFormation console** → Stacks → `wordpress-stack` → Events tab.

### 5. Get the outputs (site URL, RDS endpoint, dashboard link)

```bash
aws cloudformation describe-stacks \
  --stack-name wordpress-stack \
  --region us-east-1 \
  --query 'Stacks[0].Outputs' \
  --output table
```

---

## Alternative: Deploying via the AWS Console

1. Open **CloudFormation** in the AWS Console → **Create stack** → "With new resources (standard)".
2. Upload `wordpress-infra.yaml` as the template.
3. Fill in the parameters (Availability Zones, Key Pair name, your IP for SSH, DB username/password).
4. Check the box acknowledging IAM resource creation (`CAPABILITY_IAM`).
5. Click **Create stack** and wait for status `CREATE_COMPLETE`.
6. Go to the **Outputs** tab to find `WordPressURL`, `RDSEndpoint`, and `DashboardURL`.

---

## Using the deployed site

1. Open the `WordPressURL` output value in your browser (e.g. `http://wordpress-stack-alb-xxxxxx.us-east-1.elb.amazonaws.com`).
2. You'll land on the WordPress installation wizard — choose a language, then set:
   - Site title
   - Admin username / password
   - Admin email
3. Click **Install WordPress**, then log in at `<WordPressURL>/wp-admin`.
4. Because instances sit behind the ALB and in an Auto Scaling Group, if the ASG ever launches a **new** instance (e.g. after scaling out or replacing an unhealthy one), that instance runs the same User Data script and connects to the **same shared RDS database** — so your content, users, and settings persist across instances. (Uploaded media files are stored locally on each instance's disk by default; for a production setup you'd add EFS or S3 offload for uploads — not included in this template.)

### Viewing the CloudWatch dashboard

Open the `DashboardURL` output value, or in the console go to **CloudWatch → Dashboards → `wordpress-dashboard`**. It shows:
- **EC2 CPUUtilization** — average across the Auto Scaling Group
- **RDS CPUUtilization** — for the MySQL instance
- **ALB RequestCount** — total requests hitting the load balancer

### Scaling behavior

The Auto Scaling Group is configured with:
- Minimum: 2 instances
- Maximum: 3 instances
- Desired: 2 instances (default)

It launches instances across both configured public subnets/AZs for high availability, and only registers healthy instances (per ALB health checks on `/`) with the target group.

---

## Updating the stack

If you change the template or want to adjust parameters (e.g. bump `AsgMaxSize`):

```bash
aws cloudformation update-stack \
  --stack-name wordpress-stack \
  --template-body file://wordpress-infra.yaml \
  --region us-east-1 \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=AsgMaxSize,ParameterValue=4 \
    ... (repeat all other parameters, or use --use-previous-value for unchanged ones)
```

---

## Tearing everything down

```bash
aws cloudformation delete-stack --stack-name wordpress-stack --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name wordpress-stack --region us-east-1
```

Note: the RDS instance has `DeletionPolicy: Snapshot`, so AWS will automatically take a final snapshot before deleting it — you won't lose your database data, but the snapshot will incur storage costs until you delete it manually from the RDS console.

---

## Cost awareness

This stack is **not fully free-tier** in all configurations:
- 2–3 × `t2.micro` EC2 instances — free tier covers 750 hrs/month of **one** t2.micro, so running 2+ continuously will incur charges after the free tier hours are used.
- `db.t3.micro` RDS — check current free-tier eligibility for RDS in your account.
- ALB — charged hourly plus per-LCU, not free-tier eligible.
- Remember to delete the stack when you're done experimenting to avoid ongoing charges.

---

## Architecture summary

![WordPress on AWS architecture: VPC across two Availability Zones with an Application Load Balancer, an Auto Scaling group of WordPress EC2 instances in the public subnets, RDS MySQL in the private subnets, CloudWatch, Systems Manager, and the GitHub Actions deployment path](docs/architecture.png)

The diagram is drawn with the official AWS Architecture Icons. To edit it, open [`docs/architecture.drawio`](docs/architecture.drawio) in [draw.io](https://app.diagrams.net/) (File → Open from → Device), then export a new PNG over `docs/architecture.png` (File → Export as → PNG, zoom 200%). The icons are embedded, so it also works offline. Numbered badges match the "Request flow" legend inside the diagram.

Traffic flow: **Internet → ALB (public subnets) → WordPress EC2 instances (ASG, public subnets) → RDS MySQL (private subnets)**.
