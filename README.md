# Scotty Deploy Action

> GitHub Action for deploying docker-compose applications to Scotty (micro-PaaS)

Deploy your applications to [Scotty](https://github.com/factorial-io/scotty) with a simple GitHub Actions workflow. Scotty automatically handles SSL certificates, routing, and service discovery.

## Quick Start

```yaml
- uses: factorial-io/scotty-deploy-action@v1
  with:
    scotty-server: ${{ secrets.SCOTTY_SERVER }}
    scotty-token: ${{ secrets.SCOTTY_TOKEN }}
    app-name: my-app
    service: web:80
```

That's it! Your app is now deployed with automatic HTTPS, routing, and monitoring.

## Prerequisites

1. **Scotty Server**: Access to a Scotty server instance
2. **GitHub Secrets**: Configure these in your repository settings:
   - `SCOTTY_SERVER`: URL of your Scotty server (e.g., `https://scotty.example.com`)
   - `SCOTTY_TOKEN`: Authentication token for Scotty
3. **docker-compose.yml**: A valid compose file in your repository

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `scotty-server` | ✅ | - | URL of the Scotty server |
| `scotty-token` | ✅ | - | Access token for authentication |
| `app-name` | ✅ | - | Unique name for the application |
| `service` | * | - | Service(s) to expose (format: `SERVICE:PORT`). Multi-line supported. Required unless using `app-blueprint`. |
| `action` | | `create` | Action to perform: `create` or `destroy` |
| `folder` | | `.` | Path to folder containing compose.yml |
| `app-blueprint` | | | Use a pre-configured blueprint instead of services |
| `ttl` | | `24h` | Time to live (e.g., `2h`, `30m`, `1d`, `7d`, `forever`) |
| `basic-auth` | | | HTTP basic authentication (`USERNAME:PASSWORD`) |
| `allow-robots` | | `false` | Allow search engine indexing |
| `destroy-on-ttl` | | `false` | Destroy app after TTL instead of stopping it |
| `custom-domain` | | | Custom domain(s) (format: `DOMAIN:SERVICE`). Multi-line supported. |
| `env` | | | Environment variable(s) (format: `KEY=VALUE`). Multi-line supported. |
| `env-file` | | | Path to environment file |
| `registry` | | | Private registry name (must be configured on server) |
| `middleware` | | | Traefik middleware name(s). Multi-line supported. |
| `scottyctl-version` | | `next` | Docker image tag for scottyctl |

\* Required unless `app-blueprint` is specified

## Outputs

| Output | Description |
|--------|-------------|
| `app-url` | Deployed application URL(s) - multi-line string with one URL per line |
| `app-status` | Application status (e.g., `created`, `destroyed`) |
| `app-name` | Application name (echo of input) |

## Examples

### Review App (PR Lifecycle)

Deploy on PR open, destroy on PR close:

```yaml
name: Review App
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: factorial-io/scotty-deploy-action@v1
        with:
          scotty-server: ${{ secrets.SCOTTY_SERVER }}
          scotty-token: ${{ secrets.SCOTTY_TOKEN }}
          app-name: myapp-pr-${{ github.event.pull_request.number }}
          service: web:80
          ttl: 7d
          destroy-on-ttl: true

  cleanup:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: factorial-io/scotty-deploy-action@v1
        with:
          scotty-server: ${{ secrets.SCOTTY_SERVER }}
          scotty-token: ${{ secrets.SCOTTY_TOKEN }}
          app-name: myapp-pr-${{ github.event.pull_request.number }}
          action: destroy
```

### Multiple Services

Expose both frontend and API:

```yaml
- uses: factorial-io/scotty-deploy-action@v1
  with:
    scotty-server: ${{ secrets.SCOTTY_SERVER }}
    scotty-token: ${{ secrets.SCOTTY_TOKEN }}
    app-name: my-fullstack-app
    service: |
      frontend:80
      api:3000
```

### Custom Domain

Map custom domains to services:

```yaml
- uses: factorial-io/scotty-deploy-action@v1
  with:
    scotty-server: ${{ secrets.SCOTTY_SERVER }}
    scotty-token: ${{ secrets.SCOTTY_TOKEN }}
    app-name: my-app
    service: |
      web:80
      api:3000
    custom-domain: |
      www.example.com:web
      api.example.com:api
    allow-robots: true
```

### Environment Variables

Pass environment variables to your application:

```yaml
- uses: factorial-io/scotty-deploy-action@v1
  with:
    scotty-server: ${{ secrets.SCOTTY_SERVER }}
    scotty-token: ${{ secrets.SCOTTY_TOKEN }}
    app-name: my-app
    service: web:3000
    env: |
      NODE_ENV=production
      API_KEY=${{ secrets.API_KEY }}
      DATABASE_URL=${{ secrets.DATABASE_URL }}
```

### Basic Authentication

Protect your app with HTTP basic auth:

```yaml
- uses: factorial-io/scotty-deploy-action@v1
  with:
    scotty-server: ${{ secrets.SCOTTY_SERVER }}
    scotty-token: ${{ secrets.SCOTTY_TOKEN }}
    app-name: my-app
    service: web:80
    basic-auth: ${{ secrets.BASIC_AUTH }}  # Format: username:password
```

### Using App Blueprints

Use a pre-configured blueprint:

```yaml
- uses: factorial-io/scotty-deploy-action@v1
  with:
    scotty-server: ${{ secrets.SCOTTY_SERVER }}
    scotty-token: ${{ secrets.SCOTTY_TOKEN }}
    app-name: my-app
    app-blueprint: nodejs-app
```

## How Scotty Works

[Scotty](https://scotty.factorial.io) is a micro-PaaS that manages docker-compose-based applications with automatic routing and lifecycle management.

**What Scotty does for you:**

✅ **Traefik Configuration**: Automatically configures Traefik reverse proxy for routing traffic to your services
✅ **SSL Certificates**: Automatic HTTPS via Traefik with Let's Encrypt
✅ **Web UI**: Provides a user interface to control, inspect, and manage your running applications
✅ **Lifecycle Management**: Handles app creation, starting, stopping, and destruction with TTL-based auto-stopping

### What NOT to Include in docker-compose.yml

❌ **Port Mappings** (`ports:`): Scotty exposes services automatically
❌ **Traefik Labels**: Scotty adds routing labels automatically
❌ **VIRTUAL_HOST**: Not needed - Scotty configures routing

### Example docker-compose.yml

```yaml
version: '3.8'
services:
  web:
    image: nginx:alpine
    volumes:
      - ./public:/usr/share/nginx/html:ro

  # No ports, no labels, no virtual host needed!
```

See [examples/](examples/) for more complete examples.

## Troubleshooting

### Authentication Failed

**Error**: `401 Unauthorized`

**Solution**: Verify your `SCOTTY_TOKEN` is correct and has not expired.

### Service Not Found

**Error**: `Service 'web' not found in compose.yml`

**Solution**: Ensure the service name in `service` input matches a service in your compose.yml.

### Compose File Not Found

**Error**: `compose.yml not found`

**Solution**: Check the `folder` input points to the correct directory containing your compose.yml.

### Multiple URLs in Output

When exposing multiple services, `app-url` output contains multiple URLs (one per line):

```yaml
- name: Deploy
  id: deploy
  uses: factorial-io/scotty-deploy-action@v1
  with:
    service: |
      frontend:80
      api:3000

- name: Show URLs
  run: |
    echo "Frontend: $(echo "${{ steps.deploy.outputs.app-url }}" | sed -n '1p')"
    echo "API: $(echo "${{ steps.deploy.outputs.app-url }}" | sed -n '2p')"
```

## Links

- [Scotty Documentation](https://github.com/factorial-io/scotty)
- [scottyctl CLI Reference](https://github.com/factorial-io/scotty/blob/next/docs/content/cli.md)
- [Report Issues](https://github.com/factorial-io/scotty-deploy-action/issues)

## License

MIT - see [LICENSE](LICENSE)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)
