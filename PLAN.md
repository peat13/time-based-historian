# Docker Compose Cleanup Plan

## Goal
Create a beginner-friendly, best-practice docker-compose.yml that serves as an entry point for new Timebase users while demonstrating Docker best practices.

## Key Requirements
1. **Entry point for beginners** - Simple, clear, minimal complexity
2. **Easy collector scaling** - Use x-templates to make adding multiple collectors trivial
3. **No health checks** - Timebase services don't support health checks
4. **Keep it simple** - Avoid advanced features that confuse first-time Docker users
5. **Follow best practices** - But only those that add value without complexity

## Design Decisions

### Extension Fields (x-templates)
Use Docker Compose extension fields for reusability:
- `x-collector-template` - Complete collector definition
  - Makes it easy to add collector-2, collector-3, etc.
  - Just copy the template and override container_name, hostname, ports
- `x-timebase-common` (optional) - Shared settings if needed
- **No nesting** - Each x-template must be standalone (Docker limitation)

### Service Structure
```
services:
  ├── historian (core service - required)
  ├── explorer (UI - required for visualization)
  ├── collector (uses x-collector-template)
  ├── collector-2 (commented example showing how to add more)
  ├── node-red (optional integration service)
  └── mqtt-broker (optional message broker)
```

### Volume Strategy
- Use **named volumes** (not bind mounts)
- Simple naming: `timebase-historian-data`, `timebase-collector-data`, etc.
- Better for portability and Docker management
- Document how to find volumes with `docker volume ls`

### Environment Variables
- All configurable values use `${VAR:-default}` syntax
- Create comprehensive `.env.example` file
- Document each variable with comments
- Sensible defaults for quick start

### Networking
- Single bridge network: `timebase-network`
- Simple subnet configuration
- Services communicate via hostnames

### What to REMOVE
- ❌ Health checks (Timebase doesn't support)
- ❌ Resource limits (adds complexity, optional for advanced users)
- ❌ User/security configs (can add later, not critical for getting started)
- ❌ Hardcoded Portainer paths (use portable volumes)
- ❌ Inline Mosquitto config (use volume-mounted config file)

### What to ADD
- ✅ x-collector-template for easy scaling
- ✅ Clear section headers and comments
- ✅ .env.example with all variables documented
- ✅ Commented examples (collector-2, collector-3)
- ✅ Quick start instructions in comments
- ✅ Links to Timebase documentation
- ✅ Version pinning guidance (latest vs specific versions)

## File Structure
```
project/
├── docker-compose.yml       # Main compose file (cleaned up)
├── .env.example             # All environment variables with defaults
├── .env                     # User's actual config (gitignored)
├── PLAN.md                  # This file
└── README.md                # Quick start guide (to be updated)
```

## Implementation Steps
1. ✅ Design simplified structure
2. 🔄 Create x-common templates (no nesting)
3. ⏳ Remove health checks
4. ⏳ Simplify resource configuration
5. ⏳ Configure simple named volumes
6. ⏳ Add beginner-friendly comments
7. ⏳ Create .env.example with sensible defaults
8. ⏳ Add README quick start section

## Example: Adding a Second Collector

**Current approach (hard):**
```yaml
# Copy entire collector service definition
# Manually update every field
# Easy to miss something
```

**New approach (easy):**
```yaml
collector-2:
  <<: *x-collector-template
  container_name: collector-2
  hostname: collector-2
  ports:
    - "4522:4521"
  volumes:
    - collector-2-data:/collector
```

## Documentation References
- [Timebase Knowledge Base](https://timebase.flow-software.com/en/knowledge-base)
- [Docker Compose Extension Fields](https://docs.docker.com/compose/compose-file/#extension)
- [Docker Compose Best Practices](https://docs.docker.com/compose/production/)

## Success Criteria
- [ ] First-time Docker user can run `docker-compose up -d` successfully
- [ ] Clear comments explain every section
- [ ] Adding a second collector takes < 10 lines of YAML
- [ ] All configurable values in .env.example
- [ ] No unnecessary complexity
- [ ] Follows Docker and Timebase best practices
