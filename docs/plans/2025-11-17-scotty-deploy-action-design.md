# Scotty Deploy GitHub Action - Design Document

**Date:** 2025-11-17
**Status:** Approved

## Overview

A production-ready GitHub composite action for deploying docker-compose applications to Scotty (a micro-PaaS). The action wraps the scottyctl CLI tool available as a Docker image.

## Design Decisions

### 1. Multi-line String Inputs
**Decision:** Use multi-line strings for list-based parameters (service, custom-domain, env, middleware).

**Rationale:**
- More readable for multiple values
- GitHub Actions native pattern
- Easy to parse (split on newlines)

**Example:**
```yaml
service: |
  web:80
  api:8080
```

### 2. Output Handling
**Decision:** Capture multiple URLs as multi-line string output, parse status best-effort.

**Rationale:**
- Multiple services create multiple URLs
- Multi-line format is natural for GitHub Actions
- Users can iterate over lines in subsequent steps
- Show full scottyctl output for transparency
- Parse status opportunistically but don't fail if unparseable

**Outputs:**
- `app-url`: Multi-line string with all URLs (one per line)
- `app-status`: Parsed from output or 'created'/'unknown' fallback
- `app-name`: Echo of input for workflow convenience

### 3. Workspace Mounting
**Decision:** Mount entire workspace to /workspace, set working directory to subfolder.

**Approach:**
```bash
docker run --rm \
  -v "$GITHUB_WORKSPACE:/workspace" \
  -w "/workspace/${{ inputs.folder }}" \
  ghcr.io/factorial-io/scottyctl:next \
  app:create ...
```

**Rationale:**
- scottyctl naturally finds compose.yml in current directory
- No need for complex path manipulation
- Clean and simple

### 4. Optional Parameter Handling
**Decision:** Explicit checks for all optional parameters, special handling for booleans.

**Approach:**
```bash
# String parameters
if [ -n "${{ inputs.basic-auth }}" ]; then
  CMD="$CMD --basic-auth '${{ inputs.basic-auth }}'"
fi

# Boolean parameters
if [ "${{ inputs.allow-robots }}" = "true" ]; then
  CMD="$CMD --allow-robots"
fi
```

**Rationale:**
- Most robust and explicit
- Handles different parameter types appropriately
- Clear intent, easy to debug
- Doesn't assume scottyctl's behavior with empty values

### 5. Input Validation
**Decision:** No validation - rely on scottyctl.

**Rationale:**
- Keep it simple (per requirements)
- GitHub Actions validates required inputs
- scottyctl provides error messages for invalid values
- Avoid duplicating validation logic

## Architecture

### Action Interface (action.yml)

**Type:** Composite action

**Inputs:**
- **Required:** scotty-server, scotty-token, app-name, service
- **Optional:** action, folder, app-blueprint, ttl, basic-auth, allow-robots, destroy-on-ttl, custom-domain, env, env-file, registry, middleware, scottyctl-version
- All list-based inputs accept multi-line strings
- Booleans use string type with 'true'/'false' values

**Outputs:**
- `app-url`: Multi-line string with deployed URLs
- `app-status`: App status from scottyctl
- `app-name`: Echo of input app-name

**Steps:**
1. Pull Docker image
2. Create app (if action == 'create')
3. Destroy app (if action == 'destroy')

### Command Building Logic

**Create Command:**
1. Start with base docker run command
2. Mount workspace and set working directory
3. Add authentication via environment variables
4. Parse multi-line inputs line-by-line
5. Add optional parameters with explicit checks
6. Capture and display full output
7. Extract URLs with grep (all https:// lines)
8. Parse status with grep/awk (fallback to 'created')
9. Set GitHub Action outputs

**Destroy Command:**
1. Simple `app:destroy` call
2. No output parsing needed

## Documentation Structure

### README.md
1. **Quick Start** - Minimal 30-second example
2. **Prerequisites** - What users need
3. **Inputs Reference** - Complete table
4. **Outputs Reference** - Output descriptions
5. **Examples** - Real-world scenarios:
   - Simple review app
   - Multiple services
   - Custom domains
   - Environment variables
   - Blueprints
   - TTL with auto-destroy
   - Basic auth
6. **How Scotty Works** - What's automatic
7. **Troubleshooting** - Common issues
8. **Links** - External resources

### Supporting Files
- **LICENSE** - MIT License
- **CONTRIBUTING.md** - Contribution guidelines
- **CHANGELOG.md** - Version history

## Examples

### Workflow Examples (.github/workflows/)
1. **review-app.yml** - PR lifecycle (deploy/destroy)
2. **multi-environment.yml** - Staging/production
3. **multiple-services.yml** - Frontend + backend
4. **custom-domain.yml** - Custom domain mapping

### Compose Examples (examples/)
1. **docker-compose.simple.yml** - Single nginx service
2. **docker-compose.fullstack.yml** - Node.js + PostgreSQL
3. **docker-compose.multi-service.yml** - Multi-tier app
4. **docker-compose.build.yml** - Build context example

**Key principles:**
- NO port mappings (Scotty handles this)
- NO Traefik labels (Scotty adds these)
- NO VIRTUAL_HOST variables (not needed)
- Only essential configuration

## File Structure

```
scotty-deploy-action/
├── action.yml                      # Main action definition
├── README.md                       # Complete documentation
├── LICENSE                         # MIT License
├── CONTRIBUTING.md                 # Contribution guidelines
├── CHANGELOG.md                    # Version history
├── .github/
│   └── workflows/
│       ├── review-app.yml
│       ├── multi-environment.yml
│       ├── multiple-services.yml
│       └── custom-domain.yml
└── examples/
    ├── docker-compose.simple.yml
    ├── docker-compose.fullstack.yml
    ├── docker-compose.multi-service.yml
    └── docker-compose.build.yml
```

## Implementation Notes

### scottyctl Command Reference
Based on https://github.com/factorial-io/scotty/blob/next/docs/content/cli.md

**Authentication:**
- `SCOTTY_SERVER` environment variable
- `SCOTTY_ACCESS_TOKEN` environment variable

**Create command structure:**
```bash
scottyctl app:create <APP_NAME> --folder <FOLDER> \
  --service <SERVICE:PORT> [--service ...] \
  [--app-blueprint <BLUEPRINT>] \
  [--ttl <LIFETIME>] \
  [--basic-auth <USERNAME:PASSWORD>] \
  [--allow-robots] \
  [--destroy-on-ttl] \
  [--custom-domain <DOMAIN:SERVICE>] [--custom-domain ...] \
  [--env <KEY=VALUE>] [--env ...] \
  [--env-file <FILE>] \
  [--registry <REGISTRY>] \
  [--middleware <MIDDLEWARE>] [--middleware ...]
```

**Destroy command:**
```bash
scottyctl app:destroy <APP_NAME>
```

**Info command:**
```bash
scottyctl app:info <APP_NAME>
```

**Docker image:**
`ghcr.io/factorial-io/scottyctl:next`

### Important Constraints

**DO NOT:**
- Validate compose files (scottyctl does this)
- Check if app exists before creating (scottyctl handles this)
- Add complex error handling (keep it simple)
- Include port mappings, Traefik labels, or VIRTUAL_HOST in examples

**DO:**
- Support multi-line input for lists
- Mount workspace correctly
- Use environment variables for authentication
- Show full scottyctl output to users
- Make service required UNLESS app-blueprint is provided

## Success Criteria

- [ ] Action works with real scottyctl:next Docker image
- [ ] Users can deploy with minimal configuration (server, token, app-name, service)
- [ ] All scottyctl parameters are supported
- [ ] Documentation is clear and comprehensive
- [ ] Examples are copy-paste ready
- [ ] No unnecessary validation or complexity
- [ ] Multiple services are properly handled
- [ ] URLs are correctly extracted and output
- [ ] Destroy action works reliably

## Next Steps

Ready to proceed with implementation. Recommended approach:
1. Create isolated git worktree for development
2. Write detailed implementation plan with file-by-file tasks
3. Implement incrementally with code review checkpoints
