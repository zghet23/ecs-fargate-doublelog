# Fluent Bit Log Router — ECS Fargate + Datadog + CloudWatch

Custom Fluent Bit sidecar for AWS ECS Fargate that routes container logs to both AWS CloudWatch Logs and Datadog simultaneously, with Datadog APM integration.

## Background — Why is this needed?

### The logging problem in containers

When a process runs on a traditional server, its logs land in files on disk and a local agent (e.g., the Datadog Agent) tails those files. In containerized environments the contract is different: **containers write logs to stdout/stderr**, and the container runtime captures those streams. The operating system never exposes them as normal files to sibling processes, so a traditional file-tail agent cannot collect them without special support.

AWS ECS Fargate makes this even stricter: you have no access to the underlying host, so you cannot install or run anything outside the containers you declare in your task definition. You only control what runs *inside* the task.

### FireLens — AWS's solution

AWS built **FireLens**, a log routing mechanism for ECS, to solve this. FireLens works by intercepting stdout/stderr at the container runtime level before the logs reach the default destination (CloudWatch). It wraps each log line in a structured JSON envelope (adding ECS metadata such as cluster, task ARN, container name, etc.) and forwards it to a sidecar container that acts as a log router.

The sidecar must be a **Fluent Bit** or Fluentd process. The application container declares `awsfirelens` as its log driver — after that, every line it prints to stdout/stderr is sent directly to the Fluent Bit sidecar over a local Unix socket instead of going to the Docker log driver.

```
stdout/stderr
      │
      │  (intercepted by FireLens, wrapped in JSON)
      ▼
Fluent Bit sidecar (runs inside the same ECS task)
      │
      ├── Output plugin A  →  CloudWatch Logs
      └── Output plugin B  →  Datadog HTTP intake
```

### Why Fluent Bit and not Fluentd?

Fluent Bit is the lightweight sibling of Fluentd. It was designed for resource-constrained environments (IoT, embedded systems) and later became the standard for containerized workloads:

| | Fluent Bit | Fluentd |
|---|---|---|
| Memory footprint | ~450 KB | ~40 MB |
| CPU usage | Very low | Moderate |
| Plugin ecosystem | Smaller but covers major destinations | Very large |
| Written in | C | Ruby |

For a sidecar that only needs to forward logs to two destinations (CloudWatch and Datadog), Fluent Bit is the obvious choice — it adds virtually no overhead to the task.

### Why route to two destinations?

- **CloudWatch Logs** is the native AWS destination. It requires zero extra credentials (the task's IAM role provides access), and it is the de-facto standard for audit trails, Lambda integrations, and AWS-native alerting.
- **Datadog** provides richer querying, dashboards, anomaly detection, and — critically — **log-to-trace correlation**. When APM is enabled, Datadog can link a specific log line to the exact distributed trace that produced it, which is impossible in CloudWatch.

Sending to both simultaneously means you never have to choose: AWS operations teams keep using CloudWatch, while developers use Datadog for debugging.

### The sidecar pattern

A **sidecar** is a secondary container that runs alongside the main application container inside the same task (or pod, in Kubernetes). It shares the same network namespace, which means they communicate over `localhost` — no network hop, no latency, no external dependency. FireLens uses this pattern: the log router sidecar is always co-located with the application it serves.

This is why the three-container design works:

```
┌─────────────────────────── ECS Task (shared network) ────────────────────────────┐
│                                                                                    │
│  ┌─────────────────┐     logs      ┌──────────────────┐                           │
│  │   Application   │ ─────────────► │   Fluent Bit     │ ──► CloudWatch           │
│  │   Container     │  (FireLens)   │   Log Router     │ ──► Datadog Logs         │
│  └────────┬────────┘               └──────────────────┘                           │
│           │ APM traces / metrics                                                   │
│           │ (localhost:8126 / :8125)                                               │
│           ▼                                                                        │
│  ┌──────────────────┐                                                              │
│  │  Datadog Agent   │ ──────────────────────────────────────► Datadog APM / Metrics│
│  └──────────────────┘                                                              │
└────────────────────────────────────────────────────────────────────────────────────┘
```

The Datadog Agent handles **traces and metrics** (APM, DogStatsD); Fluent Bit handles **logs**. Both sidecars are necessary because the Datadog Agent's log collection mechanism also relies on file-tailing and cannot intercept FireLens streams directly.

---

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
