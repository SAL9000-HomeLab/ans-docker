# Changelog

All notable changes to this project are documented here. Releases are cut by
pushing a `vX.Y.Z` tag; the release workflow publishes the matching section.

## [Unreleased]

- Added: `docker_engine` role: installs Docker CE on Rocky Linux 9/10, manages `daemon.json`, opens the swarm
  ports in firewalld and installs `nfs-utils`.
- Added: `docker_swarm` role: creates the swarm and keeps it in line with the `docker_swarm_managers`,
  `docker_swarm_workers` and `docker_swarm_remove` inventory groups (join, promote, demote, drain, remove), plus
  node labels and shared overlay networks.
- Added: `arcane` role: deploys Arcane as a swarm stack, with its secrets stored as Docker secrets.
- Added: `arcane_postgres_enabled` runs PostgreSQL as a `db` service in the Arcane stack, so Arcane can use NFS
  storage and fail over between managers. Its password comes from the AWX credential.
- Added: `site.yml`, a placeholder inventory and group vars, and ansible-lint config.
