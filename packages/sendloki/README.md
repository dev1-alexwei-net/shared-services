# sendloki

A one-shot **boot event publisher** to [Grafana Loki](https://grafana.com/oss/loki/). Designed to run as an **ECS sidecar container** (or any short-lived container) that fires a single log entry on startup — so you get a record of every task boot, with the service's private IP and optional URL components — and then exits.

- **Image:** `ghcr.io/dev1-alexwei-net/shared-services/sendloki`
- **Implementation:** Bash · curl · jq (non-essential, no daemon)
- **Pushes to:** `POST <LOKI_URL>/loki/api/v1/push`

## How it works

On startup `sendloki` does three things:

1. **Reads environment variables** to build the log line.
2. **Discovers the private IP** of the host it runs on:
   - If `PRIVATE_IP` is set, that value wins.
   - Otherwise, in ECS: it queries the ECS task metadata (`169.254.170.2`) to get the container's private IP. If `ECS_NETWORK_MODE=host`, it falls back to the EC2 instance primary private IP (IMDSv2, `169.254.169.254`).
   - If not in ECS, it falls back to the hostname.
3. **Formats a Loki log entry** with `jq` (line + timestamp + private IP + any configured URL components) and **pushes it** to Loki via `POST /loki/api/v1/push`.

The push uses `curl --retry 5 --retry-delay 3` (basic auth via `LOKI_USER`/`LOKI_TOKEN` when set). The script runs once and exits `0` on success (or non-zero with the error body on stderr if the push fails after retries).

## Environment variables

| Variable | Required | Description |
| -------- | -------- | ----------- |
| `LOKI_URL` | ✅ | Base URL of the Loki endpoint, e.g. `https://loki.example.com` (trailing slash optional). |
| `SERVICE_NAME` | ✅ | Service name written into the Loki log line and the stream label `service`. |
| `LOKI_USER` | (secret) | Basic-auth username, if Loki is behind auth. Sent via `curl -u "$LOKI_USER:$LOKI_TOKEN"`. |
| `LOKI_TOKEN` | (secret) | Basic-auth password, paired with `LOKI_USER`. |
| `SERVICE_PORT` | | 1–65535. Written as a numeric `port` field in the log value. Leave blank to omit. |
| `SERVICE_PROTOCOL` | | `http` or `https`. Written as a string `protocol` field. Leave blank to omit. |
| `SERVICE_URI` | | Path beginning with `/` (e.g. `/api`). Written as a string `uri` field. Leave blank to omit. |
| `PRIVATE_IP` | | Explicit IPv4 address to use instead of auto-discovery. Takes precedence over ECS metadata and hostname. |
| `ECS_NETWORK_MODE` | | `auto` (default) uses the task ENI from ECS metadata. Set to `host` for ECS host-network tasks — uses the EC2 primary private IP via IMDSv2. |

## Example: ECS task definition

```json
{
  "taskDefinition": {
    "family": "my-service",
    "requiresCompatibilities": ["FARGATE"],
    "networkMode": "awsvpc",
    "cpu": "256",
    "memory": "512",
    "containers": [
      {
        "name": "app",
        "image": "ghcr.io/dev1-alexwei-net/shared-services/my-app",
        "essential": true
      },
      {
        "name": "sendloki",
        "image": "ghcr.io/dev1-alexwei-net/shared-services/sendloki:latest",
        "essential": false,
        "startTimeout": 30,
        "logConfiguration": {
          "logDriver": "awslogs",
          "options": {
            "awslogs-group": "/ecs/my-service",
            "awslogs-region": "us-east-2",
            "awslogs-stream-prefix": "sendloki"
          }
        },
        "environment": [
          { "name": "SERVICE_NAME", "value": "my-service" },
          { "name": "SERVICE_PORT", "value": "8080" },
          { "name": "SERVICE_PROTOCOL", "value": "http" },
          { "name": "SERVICE_URI", "value": "/api" }
        ],
        "secrets": [
          { "name": "LOKI_URL",   "valueFrom": "arn:aws:secretsmanager:us-east-2:111122223333:secret:loki/url" },
          { "name": "LOKI_USER",  "valueFrom": "arn:aws:secretsmanager:us-east-2:111122223333:secret:loki/user" },
          { "name": "LOKI_TOKEN", "valueFrom": "arn:aws:secretsmanager:us-east-2:111122223333:secret:loki/token" }
        ]
      }
    ]
  }
}
```

> **Note:** `sendloki` is marked `essential: false` so its one-shot push doesn't block the main container. If the push fails, the task still starts.

## Running locally

```bash
docker run --rm \
  -e LOKI_URL=https://loki.example.com \
  -e LOKI_USER=me \
  -e LOKI_TOKEN=topsecret \
  -e SERVICE_NAME=demo \
  -e SERVICE_PORT=8080 \
  -e SERVICE_PROTOCOL=http \
  -e SERVICE_URI=/api \
  -e PRIVATE_IP=10.0.1.25 \
  ghcr.io/dev1-alexwei-net/shared-services/sendloki:latest
```

You should see:

```
sendloki: publishing boot event for demo (10.0.1.25) ...
sendloki: published OK to https://loki.example.com
```

## Log entry shape

Each push produces a single Loki entry like:

```json
{
  "streams": [
    {
      "stream": { "service_name": "demo" },
      "values": [
        [
          "1737280000000000000",
          "{ private_ip: 10.0.1.25, timestamp: 1737280000000000000, port: 8080, protocol: \"http\", uri: \"/api\" }"
        ]
      ]
    }
  ]
}
```

The stream label is `service_name` (set to the value of `SERVICE_NAME`). The private IP is written into the log value as `private_ip` (not as a stream label, to keep the label cardinality low). The `port` field is sent as a number; `protocol` and `uri` are strings. Blank `SERVICE_*` fields are omitted from the value object.

## Tests

`tests/test_publisher.py` covers env validation, URL component rules, ECS metadata discovery, basic auth, and the push call. Run with:

```bash
PYTHONPATH=tests python -m pytest tests/test_publisher.py -q
```

## Notes

- Non-essential: mark `essential: false` in ECS so a failed push doesn't block the main container.
- One-shot: the container runs `sendloki` once at startup and exits. Use it as a sidecar, not a long-running service.
