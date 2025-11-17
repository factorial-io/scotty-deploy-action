# Scotty Deploy Action Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a production-ready GitHub composite action for deploying docker-compose applications to Scotty.

**Architecture:** Composite GitHub Action that wraps scottyctl Docker image, supports all scottyctl parameters, handles multi-line inputs for lists, and outputs deployment URLs and status.

**Tech Stack:** GitHub Actions (composite), Docker, Bash, scottyctl CLI

---

## Task 1: Create Core action.yml Structure

**Files:**
- Create: `action.yml`

**Step 1: Create action.yml with metadata and inputs**

Create the file with all inputs defined:

```yaml
name: 'Scotty Deploy'
description: 'Deploy docker-compose applications to Scotty (micro-PaaS)'
author: 'Factorial GmbH'
branding:
  icon: 'box'
  color: 'blue'

inputs:
  # Required inputs
  scotty-server:
    description: 'URL of the Scotty server'
    required: true
  scotty-token:
    description: 'Access token for Scotty authentication'
    required: true
  app-name:
    description: 'Unique name for the application'
    required: true
  service:
    description: 'Service(s) to expose publicly in format SERVICE:PORT (multi-line supported, one per line)'
    required: false

  # Action control
  action:
    description: 'Action to perform: create or destroy'
    required: false
    default: 'create'

  # Deployment configuration
  folder:
    description: 'Path to folder containing compose.yml'
    required: false
    default: '.'
  app-blueprint:
    description: 'Use a pre-configured blueprint instead of services'
    required: false
    default: ''
  ttl:
    description: 'Time to live (e.g., 2h, 30m, 1d, 7d, forever)'
    required: false
    default: '24h'

  # Access control
  basic-auth:
    description: 'HTTP basic authentication as USERNAME:PASSWORD'
    required: false
    default: ''
  allow-robots:
    description: 'Allow search engine indexing (default adds X-Robots-Tag: noindex)'
    required: false
    default: 'false'
  destroy-on-ttl:
    description: 'Destroy app after TTL instead of just stopping it'
    required: false
    default: 'false'

  # Advanced configuration
  custom-domain:
    description: 'Custom domain mapping(s) in format DOMAIN:SERVICE (multi-line supported, one per line)'
    required: false
    default: ''
  env:
    description: 'Environment variable(s) in format KEY=VALUE (multi-line supported, one per line)'
    required: false
    default: ''
  env-file:
    description: 'Path to environment file'
    required: false
    default: ''
  registry:
    description: 'Private registry name (must be configured on server)'
    required: false
    default: ''
  middleware:
    description: 'Traefik middleware name(s) (multi-line supported, one per line)'
    required: false
    default: ''

  # Tool configuration
  scottyctl-version:
    description: 'Docker image tag for scottyctl'
    required: false
    default: 'next'

outputs:
  app-url:
    description: 'Deployed application URL(s) - multi-line string with one URL per line'
    value: ${{ steps.create.outputs.app-url }}
  app-status:
    description: 'Application status'
    value: ${{ steps.create.outputs.app-status || steps.destroy.outputs.app-status }}
  app-name:
    description: 'Application name (echo of input)'
    value: ${{ inputs.app-name }}

runs:
  using: 'composite'
  steps:
    - name: Pull scottyctl Docker image
      shell: bash
      run: |
        echo "Pulling scottyctl:${{ inputs.scottyctl-version }}..."
        docker pull ghcr.io/factorial-io/scottyctl:${{ inputs.scottyctl-version }}
```

**Step 2: Commit initial action.yml structure**

```bash
git add action.yml
git commit -m "Add action.yml with inputs, outputs, and metadata"
```

---

## Task 2: Implement Create Command

**Files:**
- Modify: `action.yml`

**Step 1: Add create step to action.yml**

Add this step after the pull step (before the `runs:` closing):

```yaml
    - name: Create application
      id: create
      if: inputs.action == 'create'
      shell: bash
      run: |
        set -e

        echo "Creating Scotty application: ${{ inputs.app-name }}"

        # Build base docker command
        CMD="docker run --rm"
        CMD="$CMD -v \"$GITHUB_WORKSPACE:/workspace\""
        CMD="$CMD -w \"/workspace/${{ inputs.folder }}\""
        CMD="$CMD -e SCOTTY_SERVER=\"${{ inputs.scotty-server }}\""
        CMD="$CMD -e SCOTTY_ACCESS_TOKEN=\"${{ inputs.scotty-token }}\""
        CMD="$CMD ghcr.io/factorial-io/scottyctl:${{ inputs.scottyctl-version }}"
        CMD="$CMD app:create ${{ inputs.app-name }}"
        CMD="$CMD --folder ."

        # Add services (multi-line input)
        if [ -n "${{ inputs.service }}" ]; then
          while IFS= read -r line; do
            if [ -n "$line" ]; then
              CMD="$CMD --service \"$line\""
            fi
          done <<< "${{ inputs.service }}"
        fi

        # Add app-blueprint if specified
        if [ -n "${{ inputs.app-blueprint }}" ]; then
          CMD="$CMD --app-blueprint \"${{ inputs.app-blueprint }}\""
        fi

        # Add TTL
        if [ -n "${{ inputs.ttl }}" ]; then
          CMD="$CMD --ttl \"${{ inputs.ttl }}\""
        fi

        # Add basic auth
        if [ -n "${{ inputs.basic-auth }}" ]; then
          CMD="$CMD --basic-auth \"${{ inputs.basic-auth }}\""
        fi

        # Add boolean flags
        if [ "${{ inputs.allow-robots }}" = "true" ]; then
          CMD="$CMD --allow-robots"
        fi

        if [ "${{ inputs.destroy-on-ttl }}" = "true" ]; then
          CMD="$CMD --destroy-on-ttl"
        fi

        # Add custom domains (multi-line input)
        if [ -n "${{ inputs.custom-domain }}" ]; then
          while IFS= read -r line; do
            if [ -n "$line" ]; then
              CMD="$CMD --custom-domain \"$line\""
            fi
          done <<< "${{ inputs.custom-domain }}"
        fi

        # Add environment variables (multi-line input)
        if [ -n "${{ inputs.env }}" ]; then
          while IFS= read -r line; do
            if [ -n "$line" ]; then
              CMD="$CMD --env \"$line\""
            fi
          done <<< "${{ inputs.env }}"
        fi

        # Add env-file
        if [ -n "${{ inputs.env-file }}" ]; then
          CMD="$CMD --env-file \"${{ inputs.env-file }}\""
        fi

        # Add registry
        if [ -n "${{ inputs.registry }}" ]; then
          CMD="$CMD --registry \"${{ inputs.registry }}\""
        fi

        # Add middleware (multi-line input)
        if [ -n "${{ inputs.middleware }}" ]; then
          while IFS= read -r line; do
            if [ -n "$line" ]; then
              CMD="$CMD --middleware \"$line\""
            fi
          done <<< "${{ inputs.middleware }}"
        fi

        # Execute command and capture output
        echo "Executing: $CMD"
        OUTPUT=$(eval $CMD 2>&1)
        EXIT_CODE=$?

        echo "$OUTPUT"

        if [ $EXIT_CODE -ne 0 ]; then
          echo "Failed to create application"
          exit $EXIT_CODE
        fi

        # Extract URLs (all lines containing https://)
        URLS=$(echo "$OUTPUT" | grep -o 'https://[^ ]*' || echo "")
        echo "app-url<<EOF" >> $GITHUB_OUTPUT
        echo "$URLS" >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT

        # Extract status (best effort)
        STATUS=$(echo "$OUTPUT" | grep -i "status" | head -1 | awk '{print $NF}' || echo "created")
        echo "app-status=$STATUS" >> $GITHUB_OUTPUT

        echo "Application created successfully"
```

**Step 2: Commit create command implementation**

```bash
git add action.yml
git commit -m "Implement create command with full parameter support"
```

---

## Task 3: Implement Destroy Command

**Files:**
- Modify: `action.yml`

**Step 1: Add destroy step to action.yml**

Add this step after the create step:

```yaml
    - name: Destroy application
      id: destroy
      if: inputs.action == 'destroy'
      shell: bash
      run: |
        set -e

        echo "Destroying Scotty application: ${{ inputs.app-name }}"

        # Build docker command
        OUTPUT=$(docker run --rm \
          -e SCOTTY_SERVER="${{ inputs.scotty-server }}" \
          -e SCOTTY_ACCESS_TOKEN="${{ inputs.scotty-token }}" \
          ghcr.io/factorial-io/scottyctl:${{ inputs.scottyctl-version }} \
          app:destroy ${{ inputs.app-name }} 2>&1)

        EXIT_CODE=$?

        echo "$OUTPUT"

        if [ $EXIT_CODE -ne 0 ]; then
          echo "Failed to destroy application"
          exit $EXIT_CODE
        fi

        echo "app-status=destroyed" >> $GITHUB_OUTPUT

        echo "Application destroyed successfully"
```

**Step 2: Commit destroy command implementation**

```bash
git add action.yml
git commit -m "Implement destroy command"
```

---

## Task 4: Create Example Docker Compose Files

**Files:**
- Create: `examples/docker-compose.simple.yml`
- Create: `examples/docker-compose.fullstack.yml`
- Create: `examples/docker-compose.multi-service.yml`
- Create: `examples/docker-compose.build.yml`

**Step 1: Create simple nginx example**

```yaml
# examples/docker-compose.simple.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    volumes:
      - ./public:/usr/share/nginx/html:ro

# Note: No port mappings needed - Scotty handles this automatically
# Note: No Traefik labels needed - Scotty adds these automatically
# Note: No VIRTUAL_HOST needed - Scotty configures routing automatically
```

**Step 2: Create fullstack example**

```yaml
# examples/docker-compose.fullstack.yml
version: '3.8'

services:
  app:
    image: node:18-alpine
    working_dir: /app
    volumes:
      - ./:/app
    command: npm start
    environment:
      - DATABASE_URL=postgres://postgres:secret@db:5432/myapp
      - NODE_ENV=production
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:

# Note: Only 'app' service should be exposed publicly via --service app:3000
# Note: Database is private and only accessible within the app network
```

**Step 3: Create multi-service example**

```yaml
# examples/docker-compose.multi-service.yml
version: '3.8'

services:
  frontend:
    image: nginx:alpine
    volumes:
      - ./frontend:/usr/share/nginx/html:ro
    environment:
      - API_URL=https://myapp-api.scotty.example.com

  api:
    image: node:18-alpine
    working_dir: /api
    volumes:
      - ./api:/api
    command: npm start
    environment:
      - DATABASE_URL=postgres://postgres:secret@db:5432/api
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=api
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:

# Note: Expose both frontend:80 and api:3000 as public services
# Note: Database remains private
```

**Step 4: Create build context example**

```yaml
# examples/docker-compose.build.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        - NODE_ENV=production
    environment:
      - PORT=3000
    volumes:
      - ./data:/app/data

# Note: Scotty builds images on the server
# Note: Make sure your build context doesn't include large files
# Note: Use .dockerignore to exclude unnecessary files
```

**Step 5: Create directory and commit all examples**

```bash
mkdir -p examples
git add examples/
git commit -m "Add docker-compose example files"
```

---

## Task 5: Create Example Workflow Files

**Files:**
- Create: `.github/workflows/review-app.yml`
- Create: `.github/workflows/multi-environment.yml`
- Create: `.github/workflows/multiple-services.yml`
- Create: `.github/workflows/custom-domain.yml`

**Step 1: Create review app workflow**

```yaml
# .github/workflows/review-app.yml
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

      - name: Deploy to Scotty
        uses: factorial-io/scotty-deploy-action@v1
        with:
          scotty-server: ${{ secrets.SCOTTY_SERVER }}
          scotty-token: ${{ secrets.SCOTTY_TOKEN }}
          app-name: myapp-pr-${{ github.event.pull_request.number }}
          service: web:80
          ttl: 7d
          destroy-on-ttl: true

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Review app deployed: ${{ steps.deploy.outputs.app-url }}'
            })

  cleanup:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Destroy review app
        uses: factorial-io/scotty-deploy-action@v1
        with:
          scotty-server: ${{ secrets.SCOTTY_SERVER }}
          scotty-token: ${{ secrets.SCOTTY_TOKEN }}
          app-name: myapp-pr-${{ github.event.pull_request.number }}
          action: destroy
```

**Step 2: Create multi-environment workflow**

```yaml
# .github/workflows/multi-environment.yml
name: Multi-Environment Deploy

on:
  push:
    branches:
      - main
    tags:
      - 'v*'

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Staging
        uses: factorial-io/scotty-deploy-action@v1
        with:
          scotty-server: ${{ secrets.SCOTTY_SERVER }}
          scotty-token: ${{ secrets.SCOTTY_TOKEN }}
          app-name: myapp-staging
          service: web:80
          ttl: forever
          env: |
            ENVIRONMENT=staging
            DEBUG=true

  deploy-production:
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Production
        uses: factorial-io/scotty-deploy-action@v1
        with:
          scotty-server: ${{ secrets.SCOTTY_SERVER }}
          scotty-token: ${{ secrets.SCOTTY_TOKEN }}
          app-name: myapp-production
          service: web:80
          ttl: forever
          allow-robots: true
          env: |
            ENVIRONMENT=production
            DEBUG=false
```

**Step 3: Create multiple services workflow**

```yaml
# .github/workflows/multiple-services.yml
name: Multiple Services

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy Frontend and API
        uses: factorial-io/scotty-deploy-action@v1
        with:
          scotty-server: ${{ secrets.SCOTTY_SERVER }}
          scotty-token: ${{ secrets.SCOTTY_TOKEN }}
          app-name: myapp-dev
          service: |
            frontend:80
            api:3000
          ttl: forever
          env: |
            API_URL=https://myapp-dev-api.scotty.example.com
            FRONTEND_URL=https://myapp-dev-frontend.scotty.example.com
```

**Step 4: Create custom domain workflow**

```yaml
# .github/workflows/custom-domain.yml
name: Custom Domain

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy with Custom Domain
        uses: factorial-io/scotty-deploy-action@v1
        with:
          scotty-server: ${{ secrets.SCOTTY_SERVER }}
          scotty-token: ${{ secrets.SCOTTY_TOKEN }}
          app-name: myapp-production
          service: |
            web:80
            api:3000
          custom-domain: |
            www.example.com:web
            api.example.com:api
          ttl: forever
          allow-robots: true
          basic-auth: ${{ secrets.BASIC_AUTH }}
```

**Step 5: Create directory and commit all workflows**

```bash
mkdir -p .github/workflows
git add .github/workflows/
git commit -m "Add example workflow files"
```

---

## Task 6: Create README.md

**Files:**
- Create: `README.md`

**Step 1: Create comprehensive README**

```markdown
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

Scotty automatically handles several things for you:

✅ **SSL Certificates**: Automatic HTTPS with Let's Encrypt
✅ **Service Discovery**: Internal DNS for service-to-service communication
✅ **Load Balancing**: Traefik-based routing and load balancing
✅ **Health Checks**: Automatic container health monitoring

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
```

**Step 2: Commit README**

```bash
git add README.md
git commit -m "Add comprehensive README documentation"
```

---

## Task 7: Create Supporting Documentation

**Files:**
- Create: `LICENSE`
- Create: `CONTRIBUTING.md`
- Create: `CHANGELOG.md`

**Step 1: Create MIT License**

```
# LICENSE
MIT License

Copyright (c) 2025 Factorial GmbH

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

**Step 2: Create CONTRIBUTING.md**

```markdown
# Contributing to Scotty Deploy Action

Thank you for your interest in contributing! This document provides guidelines for contributing to the project.

## Reporting Issues

If you encounter a bug or have a feature request:

1. Check if the issue already exists in [GitHub Issues](https://github.com/factorial-io/scotty-deploy-action/issues)
2. If not, create a new issue with:
   - Clear title and description
   - Steps to reproduce (for bugs)
   - Expected vs actual behavior
   - Your environment (GitHub Actions runner, scottyctl version, etc.)
   - Relevant workflow snippets

## Submitting Pull Requests

1. **Fork the repository** and create a branch from `main`
2. **Make your changes**:
   - Follow existing code style
   - Update documentation if needed
   - Add examples if adding new features
3. **Test your changes**:
   - Test with a real Scotty server if possible
   - Verify all examples still work
4. **Commit your changes**:
   - Use clear commit messages
   - Reference issues (e.g., "Fixes #123")
5. **Submit the pull request**:
   - Describe what changed and why
   - Link related issues

## Development Setup

### Prerequisites

- Access to a Scotty server for testing
- Docker installed locally
- GitHub account

### Local Testing

Since this is a GitHub Action, local testing is limited. The best approach:

1. Create a test repository
2. Use your fork/branch in a workflow:
   ```yaml
   uses: your-username/scotty-deploy-action@your-branch
   ```
3. Test all scenarios:
   - Create application
   - Destroy application
   - Multiple services
   - Custom domains
   - Environment variables
   - Error handling

### Testing Checklist

Before submitting a PR, verify:

- [ ] Action creates apps successfully
- [ ] Action destroys apps successfully
- [ ] Multiple services work correctly
- [ ] URLs are extracted properly
- [ ] Error messages are helpful
- [ ] Documentation is updated
- [ ] Examples are still valid

## Code Style

- Use clear, descriptive variable names
- Add comments for complex logic
- Keep bash scripts readable with proper indentation
- Follow existing patterns in action.yml

## Documentation

When adding features:

- Update README.md with new inputs/outputs
- Add usage examples
- Update CHANGELOG.md
- Create example workflows if applicable

## Questions?

Open an issue or reach out to the maintainers.

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
```

**Step 3: Create CHANGELOG.md**

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial release of Scotty Deploy Action
- Support for all scottyctl parameters
- Multi-line input support for services, custom domains, env vars, and middleware
- Create and destroy actions
- Comprehensive documentation and examples
- Example workflows for common use cases
- Example docker-compose files

### Features
- 🚀 Deploy docker-compose apps to Scotty
- 🔄 PR review app lifecycle (deploy/destroy)
- 🌐 Custom domain mapping
- 🔒 HTTP basic authentication
- ⚙️ Environment variable injection
- ⏱️ TTL with optional auto-destroy
- 📊 Multi-service deployments
- 🔧 Traefik middleware support

## [1.0.0] - 2025-11-17

Initial release.
```

**Step 4: Commit all supporting files**

```bash
git add LICENSE CONTRIBUTING.md CHANGELOG.md
git commit -m "Add LICENSE, CONTRIBUTING, and CHANGELOG"
```

---

## Task 8: Create .gitignore

**Files:**
- Create: `.gitignore`

**Step 1: Create .gitignore**

```
# macOS
.DS_Store

# IDEs
.idea/
.vscode/
*.swp
*.swo

# Logs
*.log

# Secrets (should never be committed)
.env
secrets.yml
```

**Step 2: Commit .gitignore**

```bash
git add .gitignore
git commit -m "Add .gitignore"
```

---

## Final Steps

All core implementation tasks are complete. The action is ready for use with:

- ✅ Full action.yml implementation
- ✅ Create and destroy commands
- ✅ Multi-line input support
- ✅ Output parsing and extraction
- ✅ Comprehensive documentation
- ✅ Example workflows
- ✅ Example compose files
- ✅ Contributing guidelines
- ✅ MIT License

The implementation follows all design decisions:
- Multi-line strings for lists
- Best-effort output parsing
- Workspace mounting via -w flag
- Explicit parameter checks
- No unnecessary validation
- Keep it simple philosophy
