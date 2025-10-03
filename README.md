# Ansible Role: Docker Prune

![Syntax Check](https://github.com/CoffeeSprout/ansible-role-docker-prune/actions/workflows/syntax-check.yml/badge.svg)

Automated Docker cleanup using systemd timers. Prevents Docker hosts from running out of disk space by scheduling regular pruning of unused containers, images, networks, and optionally volumes.

## Requirements

- Docker installed and running
- Systemd-based Linux distribution
- Ansible 2.12 or higher

## Supported Platforms

- Debian 12 (Bookworm)
- Debian 13 (Trixie)
- AlmaLinux 8
- AlmaLinux 9
- Rocky Linux 8
- Rocky Linux 9

## Role Variables

### Basic Configuration

```yaml
# Enable/disable the prune job
docker_prune_enabled: true

# Schedule (systemd OnCalendar format)
# Default: Sunday at 6:00 AM
docker_prune_schedule: "Sun *-*-* 06:00:00"

# Random delay to avoid simultaneous execution (seconds)
# Default: 1800 (30 minutes)
# Spreads execution across hosts to reduce disk I/O stress on shared infrastructure
docker_prune_random_delay: 1800

# Catch up missed runs if system was down
docker_prune_persistent: true
```

### Pruning Options

```yaml
# Include volumes in prune (adds --volumes flag)
# Default: false (volumes are NOT pruned)
# Set to true for build systems or when you want to reclaim volume space
docker_prune_volumes: false

# Keep recent items by age filter
# Default: "24h" (keep items from last 24 hours)
# Set to "" (empty string) to DISABLE filter and prune EVERYTHING
docker_prune_filter_until: "24h"

# Additional custom filters (optional list)
docker_prune_custom_filters: []
```

## Filter Configuration Examples

### Keep Recent Items (Default)

```yaml
# Keep last 24 hours
docker_prune_filter_until: "24h"
```

**Result**: `docker system prune -af --filter "until=24h"`

### Keep Last Week

```yaml
# Keep last 7 days
docker_prune_filter_until: "168h"
```

**Result**: `docker system prune -af --filter "until=168h"`

### Prune Everything (Aggressive)

```yaml
# Prune ALL unused resources regardless of age
docker_prune_filter_until: ""
```

**Result**: `docker system prune -af` (no until filter)

⚠️ **Warning**: This will remove ALL unused images, containers, and networks. Only use on systems where you're certain older images can be safely removed.

### Build Systems (Prune Volumes)

```yaml
# Prune volumes on build systems
docker_prune_volumes: true
docker_prune_filter_until: "72h"  # Keep last 3 days
```

**Result**: `docker system prune -af --volumes --filter "until=72h"`

### Custom Filters

```yaml
# Keep images with specific label
docker_prune_filter_until: ""
docker_prune_custom_filters:
  - "label!=keep"
  - "label=temporary"
```

**Result**: `docker system prune -af --filter "label!=keep" --filter "label=temporary"`

## Schedule Configuration

The role uses systemd `OnCalendar` format for scheduling. Here are common examples:

```yaml
# Daily at 2 AM
docker_prune_schedule: "*-*-* 02:00:00"

# Every Sunday at 6 AM (default)
docker_prune_schedule: "Sun *-*-* 06:00:00"

# Every Monday at 3 AM
docker_prune_schedule: "Mon *-*-* 03:00:00"

# Every day at midnight
docker_prune_schedule: "daily"

# Every week (Sunday midnight)
docker_prune_schedule: "weekly"

# First day of every month at 4 AM
docker_prune_schedule: "*-*-01 04:00:00"
```

## Example Playbooks

### Basic Usage (Development/Testing Hosts)

```yaml
- hosts: docker_hosts
  become: true
  roles:
    - role: coffeesprout.docker-prune
```

Uses defaults: Sunday 6 AM, keep last 24 hours, no volume pruning.

### Production Environment

```yaml
- hosts: production_docker
  become: true
  roles:
    - role: coffeesprout.docker-prune
      vars:
        docker_prune_schedule: "Sun *-*-* 03:00:00"
        docker_prune_filter_until: "168h"  # Keep last week
        docker_prune_random_delay: 3600    # 1 hour spread
```

### CI/CD Build Servers

```yaml
- hosts: build_servers
  become: true
  roles:
    - role: coffeesprout.docker-prune
      vars:
        docker_prune_schedule: "daily"
        docker_prune_filter_until: "48h"   # Keep last 2 days
        docker_prune_volumes: true         # Also prune volumes
        docker_prune_random_delay: 900     # 15 minute spread
```

### Aggressive Cleanup (Staging/Demo Environments)

```yaml
- hosts: staging_docker
  become: true
  roles:
    - role: coffeesprout.docker-prune
      vars:
        docker_prune_schedule: "daily"
        docker_prune_filter_until: ""      # Prune everything!
        docker_prune_volumes: true
```

### Mixed Environment with Host-Specific Settings

```yaml
- hosts: docker_hosts
  become: true
  roles:
    - role: coffeesprout.docker-prune

# Host-specific overrides in inventory
```

**inventory/host_vars/build-server.yml:**
```yaml
docker_prune_volumes: true
docker_prune_filter_until: "72h"
```

**inventory/host_vars/production-app.yml:**
```yaml
docker_prune_filter_until: "336h"  # Keep 2 weeks
docker_prune_schedule: "Sun *-*-* 02:00:00"
```

## Operational Details

### Systemd Units

The role creates two systemd units:

- **docker-prune.service**: Oneshot service that runs the prune command
- **docker-prune.timer**: Timer that schedules the service

### Viewing Status

```bash
# Check timer status
systemctl status docker-prune.timer

# List all timers
systemctl list-timers

# Check last prune execution
systemctl status docker-prune.service

# View prune logs
journalctl -u docker-prune.service
```

### Manual Execution

```bash
# Run prune immediately (doesn't affect timer)
systemctl start docker-prune.service

# Check current Docker disk usage
docker system df
```

### Disabling Without Removing

```yaml
# Temporarily disable pruning
docker_prune_enabled: false
```

This stops and disables the timer but leaves the systemd units in place.

## Troubleshooting

### Timer Not Running

```bash
# Check if timer is enabled and active
systemctl status docker-prune.timer

# Enable and start if needed
systemctl enable --now docker-prune.timer
```

### Disk Space Still Low

```bash
# Check current Docker disk usage
docker system df

# Check if volumes are consuming space
docker system df -v

# If volumes are the issue, enable volume pruning
# (in your playbook/inventory)
```

### Prune Failed

```bash
# Check service logs
journalctl -u docker-prune.service -n 50

# Common issues:
# - Docker daemon not running
# - Insufficient disk space for Docker operations
# - Docker API version mismatch
```

### Disk I/O Stress on Shared Infrastructure

If you're experiencing disk I/O issues when many hosts prune simultaneously:

```yaml
# Increase random delay spread to distribute load
docker_prune_random_delay: 7200  # 2 hours

# Or increase filter time to prune less frequently
docker_prune_filter_until: "168h"  # Keep last week
```

## Dependencies

None. This role assumes Docker is already installed (e.g., via `geerlingguy.docker`).

## License

BSD-2-Clause

## Author Information

Created by Barry van Someren for CoffeeSprout.

Part of the CaffeineStacks managed VM service infrastructure.

## Contributing

Issues and pull requests welcome at: https://github.com/CoffeeSprout/ansible-role-docker-prune
