# Week 0 — Billing and Architecture

## Setting Up the Development Environment

For this bootcamp I'm using Ona (formerly Gitpod) as my cloud development environment.
Since the course was originally built for Gitpod Classic which shut down in October 2025,
I had to figure out the differences between the two platforms. The main change is that
environment variables are now set through the Ona dashboard as Secrets instead of 
using the `gp env` command.

Updated `.gitpod.yml` to automatically install the AWS CLI every time the environment launches:
```yaml
tasks:
  - name: aws-cli
    env:
      AWS_CLI_AUTO_PROMPT: on-partial
    init: |
      cd /workspace
      curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
      unzip awscliv2.zip
      sudo ./aws/install
      cd $THEIA_WORKSPACE_ROOT
```

## Creating an IAM User

Instead of using the Root account for everything, created a new IAM user called `david` 
with Admin access. This is best practice - you never want to use your root account for 
day to day work.

Steps taken:
- Created new user in IAM console
- Enabled console access
- Created an Admin group with AdministratorAccess policy
- Generated Access Keys for AWS CLI access
- Downloaded the CSV with credentials

## Connecting AWS CLI to My Account

Set AWS credentials as Secrets in the Ona dashboard:
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_DEFAULT_REGION

Verified the connection was working using:
aws sts get-caller-identity

Which returned my account number and confirmed I was logged in as the `david` user.

## Enabling Billing Alerts

In the Root Account went to Billing Preferences and enabled "Receive Billing Alerts" 
so AWS is allowed to send notifications when spending thresholds are hit.

## Creating a Billing Alarm with SNS and CloudWatch

First created an SNS topic which is what actually delivers the email alert:
aws sns create-topic --name billing-alarm

Then subscribed my email to that topic:
aws sns subscribe 
--topic-arn TopicARN 
--protocol email 
--notification-endpoint your@email.com

Confirmed the subscription via email.

Then created the CloudWatch alarm using a JSON config file:
aws cloudwatch put-metric-alarm --cli-input-json file://aws/json/alarm_config.json

## Creating an AWS Budget

Created a $10 monthly budget to monitor AWS spending and avoid unexpected charges.

Created two JSON configuration files:
- `aws/json/budget.json` - defines the $10 monthly budget limit
- `aws/json/budget-notifications-with-subscribers.json` - sets up email alerts at 80% threshold

First grabbed my Account ID:
aws sts get-caller-identity --query Account --output text

Then saved it as a variable and ran the budget command:
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws budgets create-budget 
--account-id $ACCOUNT_ID 
--budget file://aws/json/budget.json 
--notifications-with-subscribers file://aws/json/budget-notifications-with-subscribers.json

Verified the budget appeared in AWS Console under Billing and Cost Management showing 
"My AWS Bootcamp Budget" at $10.00 with a Healthy status.

## Lessons Learned

This was my first time using GitHub, Git, and a cloud development environment.
Here is everything I picked up along the way:

**GitHub and Git basics**
- GitHub is the permanent home of your code. Ona is just a temporary workspace.
- If you don't commit and push, work disappears when Ona archives the environment.
- The three commands to save work: `git add .` → `git commit -m "message"` → `git push`
- `git status` shows what files have changed but not yet been committed.

**How Ona works**
- Ona archives your workspace after a few days of inactivity - this is normal.
- You can restart an archived workspace and it pulls fresh from GitHub.
- This is why committing regularly matters so much.

**JSON files and CLI commands**
- JSON files are just instructions sitting there doing nothing on their own.
- You always need a CLI command to send those instructions to AWS and activate them.
- Think of it like a recipe (JSON) vs actually cooking it (CLI command).

**AWS Console vs AWS CLI**
- Both talk to the same AWS account underneath.
- You only need to do things once - pick whichever method you prefer.
- Whatever you create via CLI shows up instantly in the AWS Console and vice versa.

**The .gitignore file**
- `.gitignore` tells Git which files to completely ignore and never push to GitHub.
- My entire `aws/` folder was listed in `.gitignore` which is why it wasn't showing on GitHub.
- Had to remove `aws/` from `.gitignore` to fix this.

**Environment variables**
- Used `export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)`
  to save my account number as a variable so I didn't have to type it manually every time.
- Can verify any variable is set using `echo $VARIABLE_NAME`.
- If a variable shows blank, it means it hasn't been set yet in the current session.
