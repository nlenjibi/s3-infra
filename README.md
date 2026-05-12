# BEM13 ECS CICD — Infrastructure Repository

**Author:** Timothy Nlenjibi | **Lab:** BEM13 Running Containers on AWS

This repository contains all CloudFormation templates for deploying the BEM13 ECS CICD architecture across three environments: `dev`, `test`, and `prod`.

---

## Repository Structure

```
ecs-cicd-infra/
├── templates/           # 9 modular CloudFormation stacks
│   ├── network.yaml     # Stack 1: VPC, subnets, IGW, NAT
│   ├── security.yaml    # Stack 2: Security groups (least-privilege)
│   ├── ecr.yaml         # Stack 3: ECR repository + lifecycle rules
│   ├── iam.yaml         # Stack 4: IAM roles (OIDC, ECS, Pipeline, Deploy)
│   ├── endpoints.yaml   # Stack 5: VPC endpoints (ECR, S3, CloudWatch)
│   ├── alb.yaml         # Stack 6: ALB + blue/green target groups
│   ├── ecs.yaml         # Stack 7: ECS cluster, task, service, auto scaling
│   ├── codedeploy.yaml  # Stack 8: CodeDeploy blue/green deployment group
│   └── pipeline.yaml    # Stack 9: EventBridge + CodePipeline
├── deployments/
│   ├── dev/             # GitSync deployment files for dev (01–09)
│   ├── test/            # GitSync deployment files for test (01–09)
│   └── prod/            # GitSync deployment files for prod (01–09)
├── parameters/
│   ├── dev.json         # dev environment parameter values
│   ├── test.json        # test environment parameter values
│   └── prod.json        # prod environment parameter values (min=2 tasks)
├── deployment.yaml      # GitSync root config
└── architecture.md      # Network architecture diagrams (Mermaid)
```

---

## Stack Deployment Order

Stacks must be deployed **1 → 9** because each stack uses `Fn::ImportValue` to reference outputs from earlier stacks.

```
1. network   →  2. security  →  5. endpoints
                              →  6. alb
3. ecr       →  7. ecs       →  8. codedeploy  →  9. pipeline
4. iam       →  7. ecs
```

---

## One-Time Setup

### 1. Create the GitHub OIDC Provider in AWS IAM

```
Provider URL:  https://token.actions.githubusercontent.com
Audience:      sts.amazonaws.com
```

### 2. Update the IAM template with your GitHub repo

In `templates/iam.yaml` line 193, replace the placeholder:

```yaml
token.actions.githubusercontent.com:sub: "repo:YOUR_GITHUB_ORG/YOUR_APP_REPO:ref:refs/heads/*"
```

### 3. Deploy all 9 stacks for each environment

```bash
# Example for dev environment, stack 1
aws cloudformation deploy \
  --template-file templates/network.yaml \
  --stack-name dev-bem13-network \
  --parameter-overrides Environment=dev ProjectName=bem13 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Repeat for stacks 2–9 using the parameter values in parameters/dev.json
```

Or connect this repo to CloudFormation GitSync — it will auto-deploy on every push.

---

## Environment Differences

| Parameter | dev | test | prod |
|---|---|---|---|
| DesiredCount | 1 | 1 | 2 |
| MinCapacity | 1 | 1 | 2 |
| TaskCpu | 512 | 512 | 1024 |
| TaskMemory | 1024 | 1024 | 2048 |
| BlueTerminationWait | 5 min | 5 min | 10 min |
