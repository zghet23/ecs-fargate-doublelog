# Fluent Bit Log Router — ECS Fargate + Datadog + CloudWatch

Custom Fluent Bit sidecar for AWS ECS Fargate that routes container logs to both AWS CloudWatch Logs and Datadog simultaneously, with Datadog APM integration.

## Architecture

```
Application Container
      │
      │ (awsfirelens log driver)
      ▼
Fluent Bit Log Router (sidecar)
      ├──► AWS CloudWatch Logs
      └──► Datadog Logs

Application Container ◄──► Datadog Agent (APM / DogStatsD)
```

Three containers run in the same ECS task:

| Container | Image | Role |
|---|---|---|
| `app` | Custom ECR image | Application |
| `log-router` | Custom Fluent Bit image (this repo) | Log routing sidecar |
| `datadog-agent` | `public.ecr.aws/datadog/agent:latest` | APM + metrics |

## Repository Structure

```
.
├── Dockerfile          # Extends AWS Fluent Bit image with custom config
├── extra.conf          # Fluent Bit output configuration
└── task-definition.yml # CloudFormation ECS task definition
```

## Files

### `Dockerfile`

Extends the official AWS Fluent Bit image and copies the custom configuration:

```dockerfile
FROM public.ecr.aws/aws-observability/aws-for-fluent-bit:stable
COPY extra.conf /extra.conf
```

### `extra.conf`

Defines two output destinations:

- **CloudWatch Logs** — uses `cloudwatch_logs` plugin, log stream prefix `from-firelens-`
- **Datadog** — uses `datadog` plugin with TLS, ECS provider, service/source tagging

Both outputs use `Match *` to capture all container logs.

### `task-definition.yml`

CloudFormation template that creates an ECS task definition with:

- Fargate launch type, `awsvpc` network mode
- FireLens log driver on the application container pointing to `/extra.conf`
- Datadog Agent container with APM (port 8126) and DogStatsD (port 8125) enabled
- All sensitive values injected via environment variables

## Environment Variables

The following variables must be set on the `log-router` container at deploy time:

| Variable | Description |
|---|---|
| `CLOUDWATCH_REGION_NAME` | AWS region for CloudWatch (e.g. `us-east-1`) |
| `CLOUDWATCH_LOG_GROUP_NAME` | Target CloudWatch log group name |
| `DD_SITE` | Datadog intake site (e.g. `datadoghq.com`) |
| `DD_API_KEY` | Datadog API key |
| `DD_SERVICE` | Service name tag applied to all logs |
| `DD_SOURCE` | Source tag applied to all logs (e.g. `python`) |

The application container also needs:

| Variable | Description |
|---|---|
| `DD_AGENT_HOST` | Hostname of the Datadog agent sidecar |
| `DD_ENV` | Deployment environment (e.g. `production`) |
| `DD_SERVICE` | Service name for APM correlation |

## Build & Deploy

**Build the image:**

```bash
docker build -t fluentbit-custom .
```

**Push to ECR:**

```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker tag fluentbit-custom <account-id>.dkr.ecr.<region>.amazonaws.com/fluentbit-custom:latest
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/fluentbit-custom:latest
```

**Deploy the task definition:**

```bash
aws cloudformation deploy \
  --template-file task-definition.yml \
  --stack-name <stack-name> \
  --parameter-overrides ServiceName=<name> Cpu=256 Memory=512 ...
```

## Notes

- The `log-router` container runs as root (`User: 0`) — required to read the FireLens config file from `/extra.conf`.
- CloudWatch log groups are created automatically if they don't exist (`auto_create_group On`).
- The `DD_API_KEY` in `task-definition.yml` is a placeholder (`jkjkjkjkjk`) — replace with a real key or, preferably, inject it via AWS Secrets Manager.
