# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible role for deploying NetBox (network infrastructure management application) as a Docker Compose project. The role handles installation, configuration, plugin management, and optional custom builds of the NetBox container.

## Architecture

### Task Execution Flow

The main task flow (`tasks/main.yml`) executes in this order:
1. **validate.yml** - Validates role variables and prerequisites
2. **setup.yml** - Creates directory structure, templates configuration files
3. **plugins.yml** - Handles NetBox plugin installation from PyPI or git sources
4. **system.yml** - Sets up system-level requirements (directories, permissions)
5. **build.yml** - (Conditional) Builds custom NetBox Docker image when `netbox_docker_build: true`
6. Finally launches the Docker Compose project using `community.docker.docker_compose_v2`

### Key Components

**Configuration Templates** (`templates/`):
- `docker-compose.yml.j2` - Main orchestration file (netbox, postgres, redis, redis-cache, optional caddy)
- `Dockerfile.j2` - Custom NetBox image with plugins and extra packages
- `env/*.env.j2` - Environment variables for each service (netbox, postgres, redis, redis-cache, pgbackups)
- `configuration/extra.py.j2` - NetBox Django configuration extensions (ADMINS, logging, extra config)
- `configuration/plugins.py.j2` - NetBox plugin configuration (PLUGINS, PLUGINS_CONFIG)
- `Caddyfile.j2` - TLS reverse proxy configuration

**Build System**:
- When `netbox_docker_build: true`, clones netbox-docker repository and runs its build script
- When plugins or extra packages are specified, `netbox_force_dockerfile_build` triggers a custom Dockerfile build
- Build distinguishes between upstream image builds vs. custom Dockerfile-based builds

**Volume Management**:
- Default volumes defined in `vars/main.yml`: reports, scripts, media, static
- Additional volumes can be specified via `netbox_volumes` variable
- Volume paths map to container paths inside `/opt/netbox/netbox/`

### Important Variables

**Build Control**:
- `netbox_docker_build` - Build from netbox-docker source (full build)
- `netbox_force_dockerfile_build` - Use Dockerfile.j2 for plugins/packages
- Auto-set when `netbox_plugins` or `netbox_extra_packages` are defined

**Versions**:
- `netbox_netbox_version` - NetBox application version (e.g., 4.5.1)
- `netbox_netbox_docker_version` - netbox-docker wrapper version (e.g., 4.0.0)
- Combined as: `v{netbox_version}-{docker_version}` for image tags

**Secrets** (auto-generated with ansible.builtin.password lookup):
- `netbox_secret_key` - Django secret key (50 chars)
- `netbox_pg_password` - PostgreSQL password
- `netbox_valkey_password` - Valkey (Redis) password
- `netbox_valkey_cache_password` - Valkey cache password
- Stored in `/tmp/` with seed based on inventory_hostname

**Plugins**:
- `netbox_plugins` - List of dicts: `{name, src (optional), version (optional)}`
- `netbox_plugins_config` - Dict mapping plugin names to their config dicts
- Installed via pip or git clone during Dockerfile build

## Testing and Development

### Testing the Role

```bash
# Syntax check
ansible-playbook example-playbook.yml --syntax-check

# Run with check mode (dry run)
ansible-playbook example-playbook.yml --check

# Run against localhost
ansible-playbook example-playbook.yml -i localhost, --connection=local
```

### Local Development with TLS

For testing TLS locally, use mkcert:
```bash
mkcert -install
mkcert localhost 127.0.0.1 ::1
```

Then reference certificates in playbook vars:
```yaml
netbox_use_caddy: true
netbox_ssl_cert_bundle: /path/to/localhost+2.pem
netbox_ssl_cert_key: /path/to/localhost+2-key.pem
```

### Common Development Scenarios

**Adding a new template**: Place in `templates/`, reference in `tasks/setup.yml`

**Modifying docker-compose structure**: Edit `templates/docker-compose.yml.j2`, ensure environment files in `templates/env/` are updated

**Adding new configuration options**:
1. Add default value in `defaults/main.yml`
2. Add to appropriate template (usually `netbox.env.j2` or `configuration/extra.py.j2`)
3. Document in README.md variables table

**Testing plugin installation**:
Set `netbox_plugins` in playbook, role will automatically trigger Dockerfile build and install plugins during image build.

## File Structure

```
.
├── defaults/main.yml          # All configurable variables with defaults
├── vars/main.yml              # Internal variables (container paths, UID)
├── tasks/
│   ├── main.yml              # Orchestrates task execution
│   ├── validate.yml          # Variable validation
│   ├── setup.yml             # Template rendering, directory creation
│   ├── plugins.yml           # Plugin management
│   ├── system.yml            # System-level setup
│   └── build.yml             # Custom image building
├── templates/
│   ├── docker-compose.yml.j2 # Main compose file
│   ├── Dockerfile.j2         # Custom build with plugins
│   ├── configuration/        # NetBox configuration templates
│   │   ├── extra.py.j2      # Django extra config (ADMINS, logging)
│   │   └── plugins.py.j2    # Plugin configuration
│   ├── env/                  # Environment variable templates
│   └── *.j2                  # Other config templates
└── meta/main.yml             # Role metadata
```

## Dependencies

Required collections:
- `community.docker` - For docker_compose_v2 module
- `ansible.posix` - For mount, sysctl modules (if used)

Required on target host:
- Docker Engine
- Docker Compose V2 (as docker compose plugin)

## Branch Strategy

- `master` - Main branch (stable releases)
- `develop` - Current development branch
- PRs should target `develop` branch by default

## Git Commit Guidelines

**IMPORTANT**: Under absolutely no circumstances should code attributions be added to commit messages. Do not include any "Generated with Claude Code" or similar attributions in commits.
