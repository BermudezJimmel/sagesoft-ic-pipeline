
# AWS ECS Blue/Green Deployment Guide

## Architecture Summary

| Environment | Deployment Type | Service Name | Task Definition | Load Balancer | Notes |
|------------|----------------|--------------|----------------|---------------|------|
| **Staging** | Rolling Update | `ic-api-gateway-ecs-staging` | `ic-api-gateway-staging-task` | Same ALB / staging target group | Used to test build |
| **Production** | **Blue/Green via CodeDeploy** | `ic-api-gateway-ecs-production` | `ic-api-gateway-production-task` | ALB with **blue-tg + green-tg** | Zero downtime, safe rollback |

---

## Step 1 — Create ECR Repository

```sh
aws ecr create-repository   --repository-name ic-api-gateway-image   --region ap-southeast-1
```

---

## Step 2 — Create Cloud Map Namespace (if needed)

```sh
aws servicediscovery create-private-dns-namespace   --name insurance   --vpc vpc-xxxxxxx   --region ap-southeast-1
```

---

## Step 3 — Create Staging ECS Service (Rolling Update)

Make sure target group type = **IP**:

```sh
aws elbv2 modify-target-group-attributes   --target-group-arn YOUR_STAGING_TG_ARN   --attributes Key=targetType,Value=ip
```

Create service:

```sh
aws ecs create-service   --cluster ic-general-services-cluster   --service-name ic-api-gateway-ecs-staging   --task-definition ic-api-gateway-staging-task   --desired-count 1   --launch-type FARGATE   --service-registries registryArn=YOUR_CM_SERVICE_ARN   --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=DISABLED}"   --load-balancers "targetGroupArn=YOUR_STAGING_TG_ARN,containerName=ic-api-gateway-container,containerPort=8000"
```

---

## Step 4 — Create Production ECS Service (Blue/Green)

Do **NOT** specify task definition or target group:

```sh
aws ecs create-service   --cluster ic-general-services-cluster   --service-name ic-api-gateway-ecs-production   --desired-count 1   --deployment-controller type=CODE_DEPLOY   --service-registries registryArn=YOUR_CM_SERVICE_ARN   --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=DISABLED}"
```

---

## Step 5 — CodeDeploy Setup

Create application:
```sh
aws deploy create-application   --application-name ic-api-gateway-cd   --compute-platform ECS
```

Create Deployment Group using Console:

| Setting | Value |
|--------|-------|
| Service | `ic-api-gateway-ecs-production` |
| Load Balancer | choose ALB |
| Blue Target Group | `ic-api-gateway-blue-tg` |
| Green Target Group | `ic-api-gateway-green-tg` |
| Termination Wait Time | **10 minutes recommended** |

---

## Step 6 — Add Required Files to Repository

### `buildspec.yml`
\`\`\`yaml
version: 0.2
env:
  variables:
    APP_NAME: ic-api-gateway-image
    CONTAINER_NAME: ic-api-gateway-container

phases:
  pre_build:
    commands:
      - ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - REGION=$AWS_DEFAULT_REGION
      - ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com"
      - IMAGE_REPO="$ECR_URI/$APP_NAME"
      - IMAGE_TAG=${CODEBUILD_RESOLVED_SOURCE_VERSION:0:8}
      - aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin "$ECR_URI"

  build:
    commands:
      - docker build -t "$IMAGE_REPO:$IMAGE_TAG" -t "$IMAGE_REPO:latest" .

  post_build:
    commands:
      - docker push "$IMAGE_REPO:$IMAGE_TAG"
      - docker push "$IMAGE_REPO:latest"
      - printf '[{"name":"%s","imageUri":"%s"}]
' "$CONTAINER_NAME" "$IMAGE_REPO:$IMAGE_TAG" > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
\`\`\`

---

### `appspec.yml`
\`\`\`yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: taskdef.json
        LoadBalancerInfo:
          ContainerName: ic-api-gateway-container
          ContainerPort: 8000
\`\`\`

---

### `taskdef.json`

\`\`\`json
{
  "family": "ic-api-gateway-production-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "ic-api-gateway-container",
      "image": "PLACEHOLDER_IMAGE",
      "portMappings": [
        { "containerPort": 8000, "protocol": "tcp" }
      ]
    }
  ]
}
\`\`\`

---

## Step 7 — Pipeline

Pipeline stages:

```
Source → Build → Deploy to Staging → Manual Approval → Deploy to Production (CodeDeploy)
```

Prod deploy config:

| Field | Value |
|------|-------|
| AppSpec Template Artifact | BuildArtifact |
| AppSpec Template Path | appspec.yml |
| TaskDefinition Template Artifact | BuildArtifact |
| TaskDefinition Template Path | taskdef.json |
| Image Placeholder | `PLACEHOLDER_IMAGE` |

---

## Rollback Strategy

| State | Rollback Option |
|------|----------------|
| During wait window | **Click Stop and Roll Back** |
| After wait window | Re-deploy previous image tag |

---
