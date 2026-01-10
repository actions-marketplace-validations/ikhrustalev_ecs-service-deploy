# ECS Service Deploy

[![Test](https://github.com/ikhrustalev/ecs-service-deploy/actions/workflows/test_action.yml/badge.svg)](https://github.com/ikhrustalev/ecs-service-deploy/actions/workflows/test_action.yml)
[![Release](https://img.shields.io/github/v/release/ikhrustalev/ecs-service-deploy)](https://github.com/ikhrustalev/ecs-service-deploy/releases)
[![License](https://img.shields.io/github/license/ikhrustalev/ecs-service-deploy)](LICENSE)

A lightweight GitHub Action for redeploying ECS services without the ceremony of full task definition management.

## When to Use This

| Use Case                                | This Action      | [Official ECS Action](https://github.com/aws-actions/amazon-ecs-deploy-task-definition) |
| --------------------------------------- | ---------------- | --------------------------------------------------------------------------------------- |
| Force redeploy to pull `:latest`        | ✅ Simple        | ❌ Requires task definition file                                                        |
| Quick rollback to known task definition | ✅ Pass ARN      | ❌ Need to rebuild definition file                                                      |
| Scale service during deploy             | ✅ One parameter | ❌ Separate step needed                                                                 |
| Full CI/CD with image build             | ❌ Not the goal  | ✅ Built for this                                                                       |

**TL;DR:** Use this for simple redeploys. Use the official action for full build-register-deploy pipelines.

## Inputs

| Input                  | Description                                            | Required | Default |
| ---------------------- | ------------------------------------------------------ | -------- | ------- |
| `cluster_arn`          | ARN of the ECS cluster                                 | Yes      | —       |
| `service_name`         | Name of the ECS service                                | Yes      | —       |
| `region`               | AWS region (uses env `AWS_REGION` if not set)          | No       | —       |
| `force_new_deployment` | Force new deployment even if task definition unchanged | No       | `true`  |
| `task_definition_arn`  | Deploy specific task definition ARN                    | No       | —       |
| `desired_count`        | Set desired task count                                 | No       | —       |
| `wait`                 | Wait for service to stabilize                          | No       | `true`  |

## Outputs

| Output            | Description                            |
| ----------------- | -------------------------------------- |
| `deployment_id`   | ID of the created deployment           |
| `task_definition` | Task definition ARN used in deployment |

## Prerequisites

Configure AWS credentials before using this action:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-arn: arn:aws:iam::123456789012:role/my-role
    aws-region: us-east-1
```

Required IAM permissions:

- `ecs:UpdateService`
- `ecs:DescribeServices` (if `wait: true`)

## Examples

### Pull `:latest` and Restart (Dev/Staging)

The most common use case — your task definition points to `:latest` and you want ECS to pull the new image:

```yaml
- name: Redeploy to pull latest image
  uses: ikhrustalev/ecs-redeploy-service@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/dev-cluster
    service_name: api-service
```

### Quick Rollback

Deploy a specific task definition (useful for rollbacks):

```yaml
- name: Rollback to previous version
  uses: ikhrustalev/ecs-redeploy-service@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod-cluster
    service_name: api-service
    task_definition_arn: arn:aws:ecs:us-east-1:123456789012:task-definition/api:42
```

### Scale During Deployment

Adjust desired count while redeploying:

```yaml
- name: Scale up and redeploy
  uses: ikhrustalev/ecs-redeploy-service@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod-cluster
    service_name: api-service
    desired_count: 4
```

### Fire and Forget (No Wait)

Skip waiting for stability — useful in non-critical environments or when you have separate monitoring:

```yaml
- name: Trigger redeploy without waiting
  uses: ikhrustalev/ecs-redeploy-service@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/dev-cluster
    service_name: api-service
    wait: false
```

### Multi-Region Deploy

Deploy to a specific region:

```yaml
- name: Deploy to EU region
  uses: ikhrustalev/ecs-redeploy-service@v1
  with:
    cluster_arn: arn:aws:ecs:eu-west-1:123456789012:cluster/eu-cluster
    service_name: api-service
    region: eu-west-1
```

### Full Pipeline Example

```yaml
name: Deploy to Staging

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy-role
          aws-region: us-east-1

      - name: Build and push image
        run: |
          # Your build logic here
          docker build -t my-repo:latest .
          docker push my-repo:latest

      - name: Redeploy ECS service
        uses: ikhrustalev/ecs-redeploy-service@v1
        with:
          cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/staging
          service_name: api-service

      - name: Notify on success
        run: echo "Deployed with task definition ${{ steps.deploy.outputs.task_definition }}"
```

## License

MIT
