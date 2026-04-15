# github-actions-for-multiple-ec2-instance-using-asg-lb-ssm-oidc-pipeline
Production-ready CI/CD pipeline using GitHub Actions, OIDC, AWS SSM, Auto Scaling Group, and Application Load Balancer for zero-downtime rolling deployments.

🧭 STEP 0: What You Are Building
You will have:

GitHub Actions → CI/CD

AWS Identity and Access Management (OIDC) → Secure login

Amazon EC2 (2 instances)

AWS Auto Scaling → Maintain instances

Elastic Load Balancing (ALB) → Traffic routing

AWS Systems Manager → Deployment execution

🔷 STEP 1: Create EC2 Launch Template
Go to EC2 → Launch Templates → Create

Add:
AMI: Ubuntu / Amazon Linux

Instance type: t2.micro (free tier)

Key pair: optional

Security group:

Allow HTTP (80)

Allow SSH (optional)

📌 User Data (VERY IMPORTANT)
#!/bin/bash
apt update -y
apt install nginx git -y

systemctl start nginx
systemctl enable nginx

# Ensure SSM agent is running
snap install amazon-ssm-agent --classic || true
systemctl enable snap.amazon-ssm-agent.amazon-ssm-agent.service
systemctl start snap.amazon-ssm-agent.amazon-ssm-agent.service
🔷 STEP 2: Create IAM Role for EC2
Go to IAM → Roles → Create Role

Attach:
✅
AmazonSSMManagedInstanceCore

Attach this role to Launch Template.

🔷 STEP 3: Create Target Group
Go to EC2 → Target Groups

Configure:
Type: Instances

Protocol: HTTP

Port: 80

Health Check:
Path:
/

Interval: 15 sec

Healthy threshold: 2

🔷 STEP 4: Create Application Load Balancer
Go to EC2 → Load Balancer → Create ALB

Configure:
Listener: HTTP (80)

Select Target Group (created above)

🔷 STEP 5: Create Auto Scaling Group
Go to EC2 → Auto Scaling Groups

Configure:
Select Launch Template

Attach to Target Group

Set:

Desired = 2

Min = 2

Max = 2

👉 After creation:

2 EC2 instances will launch

Both will be registered to ALB

🔷 STEP 6: Verify Setup
Open ALB DNS → should show Nginx page

Go to:

EC2 → Instances → check both running

Target Group → both should be healthy

🔷 STEP 7: Setup OIDC (GitHub → AWS)
Go to IAM → Identity Providers

Create:
Provider URL:

https://token.actions.githubusercontent.com
Audience:

sts.amazonaws.com
🔷 STEP 8: Create OIDC IAM Role
Go to IAM → Roles → Create Role

Trusted Entity:
GitHub OIDC

Add condition:
repo:<your-username>/<repo-name>:ref:refs/heads/main
Attach Permissions:
AmazonSSMFullAccess
(or limited)

AmazonEC2ReadOnlyAccess

ElasticLoadBalancingFullAccess

🔷 STEP 9: GitHub Repository Setup
No AWS secrets needed ✅ (because of OIDC)

🔷 STEP 10: Add GitHub Actions Workflow
Create file:

.github/workflows/deploy.yml

name: Rolling Deploy

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: us-east-1
  ASG_NAME: your-asg-name
  TARGET_GROUP_ARN: your-target-group-arn

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
          aws-region: ${{ env.AWS_REGION }}

      - name: Get Instances
        id: instances
        run: |
          IDS=$(aws ec2 describe-instances \
            --filters "Name=tag:aws:autoscaling:groupName,Values=$ASG_NAME" \
            --query "Reservations[*].Instances[*].InstanceId" \
            --output text)

          echo "ids=$IDS" >> $GITHUB_OUTPUT

      - name: Rolling Deploy
        run: |
          for INSTANCE in ${{ steps.instances.outputs.ids }}
          do
            echo "Deploying on $INSTANCE"

            aws elbv2 deregister-targets \
              --target-group-arn $TARGET_GROUP_ARN \
              --targets Id=$INSTANCE

            aws elbv2 wait target-deregistered \
              --target-group-arn $TARGET_GROUP_ARN \
              --targets Id=$INSTANCE

            COMMAND_ID=$(aws ssm send-command \
              --instance-ids $INSTANCE \
              --document-name "AWS-RunShellScript" \
              --query "Command.CommandId" \
              --parameters commands='[
                "cd /var/www/html",
                "sudo rm -rf *",
                "sudo git clone https://github.com/YOUR_USERNAME/YOUR_REPO .",
                "sudo systemctl restart nginx"
              ]' \
              --output text)

            aws ssm wait command-executed \
              --command-id $COMMAND_ID \
              --instance-id $INSTANCE

            aws elbv2 register-targets \
              --target-group-arn $TARGET_GROUP_ARN \
              --targets Id=$INSTANCE

            aws elbv2 wait target-in-service \
              --target-group-arn $TARGET_GROUP_ARN \
              --targets Id=$INSTANCE

            echo "$INSTANCE deployed successfully"
          done
🔷 STEP 11: Test Deployment
Push code to
main

GitHub Actions runs

Watch logs:

Instance 1 → deploy → healthy

Instance 2 → deploy → healthy

👉 Open ALB URL → updated site

🔥 FINAL RESULT
You now have:

✅ Zero downtime deployment
✅ Secure AWS access (OIDC, no keys)
✅ No SSH required (SSM)
✅ Load-balanced traffic
✅ Auto scaling ready
