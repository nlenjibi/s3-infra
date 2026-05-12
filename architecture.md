# BEM13 ECS CICD — Network Architecture Diagram

Author: Timothy Nlenjibi | Lab: BEM13 Running Containers on AWS

---

## Full System Architecture

```mermaid
flowchart TD
    DEV["👨‍💻 Developer\npushes code"]

    subgraph GH["GitHub (App Repository)"]
        direction TB
        BRANCH["Branch: dev / test / main"]
        CI["ci.yml\nBuild + Test\n(mvn clean verify)"]
        DEPLOY["deploy.yml\nBuild JAR → Docker image\nOIDC → ECR push"]
    end

    subgraph AWS["AWS — us-east-1"]

        subgraph IAM["IAM"]
            OIDC["OIDC Provider\ntoken.actions.githubusercontent.com"]
            ROLE["GitHubActionsRole\n(temp credentials, 15 min)"]
        end

        subgraph ECR["Amazon ECR"]
            REPO["bem13-app\nTags: {env}-{sha} + {env}-latest\nLifecycle: keep 15, expire untagged 1d"]
        end

        subgraph S3["Amazon S3 — Artifact Bucket"]
            ZIP["{env}/deploy-artifacts.zip\ntaskdef.json + appspec.yaml"]
        end

        EB["Amazon EventBridge\nRule: ecr PUSH SUCCESS\ntag = {env}-latest"]

        subgraph CP["AWS CodePipeline"]
            SRC1["Source: ECR\nimageDetail.json"]
            SRC2["Source: S3\ndeploy-artifacts.zip"]
            DEPLOY_STAGE["Deploy: CodeDeployToECS\nSubstitutes IMAGE1_NAME"]
        end

        subgraph CD["AWS CodeDeploy — ECS Blue/Green"]
            BLUE["Blue Task Set\n(current version)"]
            GREEN["Green Task Set\n(new version)"]
            SHIFT["Traffic Shift\nPort 80 → Green\nBlue terminated after 5–10 min"]
        end

        subgraph VPC["VPC — 10.0.0.0/16 (Multi-AZ)"]

            subgraph PUB["Public Subnets (AZ-a: 10.0.1.0/24 | AZ-b: 10.0.2.0/24)"]
                IGW["Internet Gateway"]
                ALB["Application Load Balancer\nPort 80 → Production (Blue/Green)\nPort 9090 → Test (Green validation)"]
                NAT["NAT Gateway"]
            end

            subgraph PRIV["Private Subnets (AZ-a: 10.0.3.0/24 | AZ-b: 10.0.4.0/24)"]
                TASK_A["ECS Fargate Task\nAZ-a\nSpring Boot :8080\nSPRING_PROFILES_ACTIVE=env"]
                TASK_B["ECS Fargate Task\nAZ-b\nSpring Boot :8080\nSPRING_PROFILES_ACTIVE=env"]
            end

            subgraph ENDPOINTS["VPC Endpoints (Private Subnets)"]
                EP1["ecr.api — Interface"]
                EP2["ecr.dkr — Interface"]
                EP3["S3 — Gateway"]
                EP4["logs — Interface"]
            end

        end

        CW["Amazon CloudWatch Logs\n/ecs/{env}-bem13\n30-day retention"]

        subgraph ASG["ECS Auto Scaling"]
            SCALE["Min: 1 | Desired: 1 | Max: 4\nCPU > 70% → scale out\nCooldown: 60s out / 300s in"]
        end

    end

    INTERNET["🌐 Internet\nUsers"]

    DEV --> BRANCH
    BRANCH --> CI
    BRANCH --> DEPLOY
    DEPLOY -- "OIDC AssumeRole" --> OIDC
    OIDC --> ROLE
    ROLE --> S3
    ROLE --> ECR
    DEPLOY -- "1. Upload artifacts first" --> ZIP
    DEPLOY -- "2. Push image" --> REPO
    REPO -- "Triggers on {env}-latest" --> EB
    EB --> CP
    ZIP --> SRC2
    REPO --> SRC1
    SRC1 --> DEPLOY_STAGE
    SRC2 --> DEPLOY_STAGE
    DEPLOY_STAGE --> CD
    CD --> GREEN
    GREEN --> SHIFT
    SHIFT --> BLUE
    INTERNET --> IGW --> ALB
    ALB -- "Port 80 (prod)" --> TASK_A
    ALB -- "Port 80 (prod)" --> TASK_B
    ALB -- "Port 9090 (test)" --> GREEN
    TASK_A --> EP1 & EP2 & EP3 & EP4
    TASK_B --> EP1 & EP2 & EP3 & EP4
    EP1 & EP2 --> REPO
    EP4 --> CW
    TASK_A & TASK_B --> CW
    ASG --> TASK_A & TASK_B
```

---

## CloudFormation Stack Dependency Chain

```mermaid
flowchart LR
    N["1. network\nVPC · Subnets · IGW · NAT"]
    S["2. security\nALB SG · ECS SG · Endpoint SG"]
    E["3. ecr\nRepository + Lifecycle"]
    I["4. iam\n6 Roles (OIDC · ECS · Pipeline · Deploy · EventBridge)"]
    EP["5. endpoints\nECR API · ECR DKR · S3 · Logs"]
    A["6. alb\nALB · Blue TG · Green TG · Listeners"]
    C["7. ecs\nCluster · TaskDef · Service · AutoScaling"]
    CD["8. codedeploy\nApp · DeploymentGroup"]
    P["9. pipeline\nS3 Bucket · EventBridge · CodePipeline"]

    N --> S & EP & A
    S --> EP & A & C
    E --> C & P
    I --> C & CD & P
    EP --> C
    A --> C & CD
    C --> CD & P
    CD --> P
```

---

## Security Group Rules

```mermaid
flowchart LR
    INT["0.0.0.0/0"] -- "TCP 80" --> ALB_SG["ALB SG"]
    VPC_CIDR["VPC 10.0.0.0/16"] -- "TCP 9090" --> ALB_SG
    ALB_SG -- "TCP 8080" --> ECS_SG["ECS SG"]
    ECS_SG -- "TCP 443" --> EP_SG["Endpoint SG"]
```

---

## Three-Environment Deployment Model

```mermaid
flowchart LR
    subgraph APP["App Repository (lab2)"]
        DEV_BR["branch: dev"]
        TEST_BR["branch: test"]
        MAIN_BR["branch: main"]
    end

    subgraph INFRA["Infra Repository (ecs-cicd-infra)"]
        DEV_S["deployments/dev/\n01–09 stacks\nenv=dev"]
        TEST_S["deployments/test/\n01–09 stacks\nenv=test"]
        PROD_S["deployments/prod/\n01–09 stacks\nenv=prod"]
    end

    DEV_BR -- "image: dev-{sha}" --> DEV_ECS["dev ECS cluster\nALB: dev-bem13-alb"]
    TEST_BR -- "image: test-{sha}" --> TEST_ECS["test ECS cluster\nALB: test-bem13-alb"]
    MAIN_BR -- "image: prod-{sha}" --> PROD_ECS["prod ECS cluster\nALB: prod-bem13-alb\nmin=2 tasks"]

    DEV_S --> DEV_ECS
    TEST_S --> TEST_ECS
    PROD_S --> PROD_ECS
```
