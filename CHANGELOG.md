# Changelog

All notable changes to this project are documented here. Releases are cut by
pushing a `vX.Y.Z` tag; the release workflow publishes the matching section.

## [Unreleased]

- Added: `docker_engine` role: installs Docker CE on Rocky Linux 9/10, manages `daemon.json`, opens the swarm
  ports in firewalld and installs `nfs-utils`.
- Added: `docker_swarm` role: creates the swarm and keeps it in line with the `docker_swarm_managers`,
  `docker_swarm_workers` and `docker_swarm_remove` inventory groups (join, promote, demote, drain, remove), plus
  node labels and shared overlay networks.
- Added: `arcane` role: deploys Arcane as a swarm stack, with its secrets stored as Docker secrets. The job waits
  for the services to start and, if they don't, fails with their task errors and recent logs.
- Added: `arcane_postgres_enabled` runs PostgreSQL as a `db` service in the Arcane stack, so Arcane can use NFS
  storage and fail over between managers. Its password comes from the AWX credential.
- Added: `keepalived` role: floating IPs shared by the managers over unicast VRRP. With `track_tcp_port`, an IP
  follows a service published on host-mode ports (e.g. Nginx Proxy Manager).
- Changed: The manager and worker lists keep inventory order, so the first listed manager creates a new swarm.
- Added: `site.yml`, a placeholder inventory and group vars, and ansible-lint config.
