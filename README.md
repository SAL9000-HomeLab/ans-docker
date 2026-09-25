# ans-docker

Builds a highly available Docker Swarm on Rocky Linux 10 hosts and deploys [Arcane](https://getarcane.app/docs)
to manage it. Run `site.yml` from AWX. Each run brings the cluster in line with the inventory, so adding, removing,
promoting or demoting a node only takes an inventory change and another run.

| Role | What it does |
| --- | --- |
| [docker_engine](roles/docker_engine) | Installs Docker CE from Docker's EL repo, writes `daemon.json`, opens the swarm ports in firewalld, installs `nfs-utils` for NFS volumes |
| [docker_swarm](roles/docker_swarm) | Creates the swarm, joins/promotes/demotes/drains/removes nodes, applies node labels, creates shared overlay networks |
| [arcane](roles/arcane) | Runs Arcane as a swarm service on a manager, optionally with its own PostgreSQL database, with its secrets stored as Docker secrets |

The values committed in `inventory/` and `group_vars/` are placeholders (`example.com`). Put your real hosts in the
AWX inventory and your secrets in AWX credentials, or in local files that are gitignored (`local/`,
`extra-vars*.yml`). Never commit credentials: this repository is public.

## Inventory

The roles are driven by three groups (see [inventory/hosts.yml](inventory/hosts.yml)):

| Group | Purpose |
| --- | --- |
| `docker_swarm_managers` | Control plane. Also runs workloads unless drained. Use an odd number: 3 tolerates one failure, 5 tolerates two. |
| `docker_swarm_workers` | Run workloads only. |
| `docker_swarm_remove` | Hosts to take out of the swarm. A host here is ignored in the other two groups. |

Put all three under a parent `docker_swarm` group so [group_vars/docker_swarm.yml](group_vars/docker_swarm.yml)
applies to every node (connection user, shared networks, Arcane settings).

### Adding a node

1. Build the VM (e.g. with `proxmox-deploy` from the `tpl-rocky-10` template).
2. Add it to `docker_swarm_managers` or `docker_swarm_workers`.
3. Run the job. Limiting the job to the new host (`--limit new-host`) is fine: the play still asks the existing
   managers for the join token.

New managers join one at a time so the cluster doesn't lose quorum.

### Promoting or demoting

Move the host between `docker_swarm_managers` and `docker_swarm_workers`, then run the job.

### Removing a node

1. Add the host to `docker_swarm_remove`. It can stay in its old group until the job has run.
2. Run the job **without a limit**, or with a limit that includes at least one manager
   (`docker_swarm_managers:old-host`). Removal runs from a manager.
3. The play demotes the node if it's a manager, drains it, waits for its tasks to start elsewhere, runs
   `docker swarm leave` on it, and deletes it from the swarm.
4. Delete the host from the inventory.

If the host is already dead and Ansible can't reach it, it's deleted from the swarm once the swarm reports it
`Down`. If the swarm node name isn't the inventory name or its short form, set `docker_swarm_node_hostname`.
The play refuses to remove a node that is still `Ready` but unreachable to Ansible, because it would keep running
as a detached swarm member.

### Maintenance

Set `docker_swarm_node_availability: drain` on a host and run the job to move its workloads off. Set it back to
`active` (the default) afterwards.

### Safety checks

- The play fails, and doesn't create a second cluster, when the managers report different swarm cluster IDs, or
  when no manager is in a swarm but some manager couldn't be checked.
- A host that belongs to a different swarm is reported, not joined.
- There must be at least one manager outside `docker_swarm_remove`. An even number of managers gives a warning.

## Variables

### docker_engine

| Variable | Default | Notes |
| --- | --- | --- |
| `docker_engine_version` | `""` (latest) | Pin, e.g. `29.8.1`. |
| `docker_engine_users` | `[]` | Added to the `docker` group (root-equivalent). |
| `docker_engine_daemon_options` | `{}` | Merged over `docker_engine_daemon_defaults` (json-file logs, 10 MB × 3) into `daemon.json`. `live-restore` is rejected: swarm doesn't support it. |
| `docker_engine_extra_packages` | `[nfs-utils]` | Extra host packages. |
| `docker_engine_conflicting_packages` | podman, runc, old docker | Removed before installing (runc conflicts with `containerd.io`). |
| `docker_engine_firewalld_manage` | `true` | Opens 2377/tcp, 7946/tcp+udp, 4789/udp in `docker_engine_firewalld_zone` when firewalld is running. |
| `docker_engine_firewalld_allow_esp` | `false` | Needed for `encrypted: true` overlay networks. |

A change to `daemon.json` restarts Docker on every node in the play at once. On a running cluster, apply that
kind of change a few nodes at a time with `--limit`.

### docker_swarm

| Variable | Default | Notes |
| --- | --- | --- |
| `docker_swarm_advertise_addr` | `ansible_default_ipv4.address` | Per host. IP or interface name. |
| `docker_swarm_init_options` | `[]` | Extra `docker swarm init` args, used once, e.g. `["--default-addr-pool", "10.200.0.0/16"]`. |
| `docker_swarm_node_labels` | `{}` | Per host. Used in placement constraints (`node.labels.<key> == <value>`). |
| `docker_swarm_node_labels_exclusive` | `false` | `true` removes labels that aren't listed (including ones added in Arcane). |
| `docker_swarm_node_availability` | `active` | Per host: `active`, `pause` or `drain`. |
| `docker_swarm_node_hostname` | unset | Per host. Swarm node name, only needed to remove an unreachable host. |
| `docker_swarm_overlay_networks` | `[]` | Shared networks that stacks declare `external: true`, e.g. `proxy`. Keys: `name`, `attachable` (default `true`), `encrypted`, `subnet`, `gateway`. Networks are only created, never deleted. |
| `docker_swarm_manager_group` / `_worker_group` / `_remove_group` | the group names above | |

### arcane

| Variable | Default | Notes |
| --- | --- | --- |
| `arcane_encryption_key` | **required, secret** | `openssl rand -hex 32`, generated once. Changing it later makes Arcane's stored credentials unreadable. |
| `arcane_postgres_enabled` | `false` | Run PostgreSQL as a `db` service in the Arcane stack. See below. |
| `arcane_postgres_password` | `""` | Secret. Required with `arcane_postgres_enabled`. |
| `arcane_postgres_version` | `18-alpine` | Tag of `docker.io/postgres`. Major upgrades need a dump and restore. |
| `arcane_postgres_db` / `arcane_postgres_user` | `arcane` / `arcane` | |
| `arcane_postgres_volume_driver(_opts)` | `local` / `{}` | Volume for `/var/lib/postgresql`, e.g. NFS options. |
| `arcane_postgres_placement_constraints` | same as `arcane_placement_constraints` | Where the database may run. |
| `arcane_database_url` | `""` (SQLite) | Secret. URL of an external PostgreSQL database. Not used with `arcane_postgres_enabled`. |
| `arcane_oidc_client_secret` | `""` | Secret. |
| `arcane_admin_static_api_key` | `""` | Secret. Fixed API key for automation. |
| `arcane_version` | `v2.13.1` | Image tag of `ghcr.io/getarcaneapp/manager`. |
| `arcane_app_url` | `http://<first manager>:3552` | The URL users open. |
| `arcane_port` / `arcane_publish_mode` | `3552` / `ingress` | `ingress` answers on every node's IP. |
| `arcane_placement_constraints` | manager with label `arcane=true` | See below. |
| `arcane_data_volume_driver(_opts)` | `local` / `{}` | e.g. NFS options, like the existing stacks use. |
| `arcane_networks` | `[]` | External overlay networks to attach (e.g. a reverse-proxy network). |
| `arcane_environment` | `{}` | Non-secret settings: `TRUSTED_PROXIES`, `OIDC_ENABLED`, `OIDC_ISSUER_URL`, `OIDC_CLIENT_ID`, … |
| `arcane_deploy_wait_retries` / `_delay` | `60` / `5` | How long to wait for the services to start. If they don't, the job fails with each service's task errors and recent logs. |

Secret values are stored as Docker secrets named `arcane_<key>_<hash>` and passed to Arcane through its `*_FILE`
variables, so they never appear in the stack file, `docker service inspect`, or the job log. A changed value
creates a new secret, redeploys, and removes the old one.

#### Where Arcane runs

Arcane needs a manager node, because it drives the swarm through the local Docker socket. By default its data
(SQLite database, projects) is in a local volume, so it's pinned to the manager labelled `arcane=true`. If that
node fails, the swarm and every stack keep running; only the Arcane UI is down until the node returns.

To let Arcane fail over to any manager, it needs shared storage and PostgreSQL instead of SQLite (keep SQLite off
NFS: its locking isn't reliable there). The simplest way is the bundled database:

```yaml
arcane_postgres_enabled: true            # arcane_postgres_password comes from the AWX credential
arcane_placement_constraints:
  - node.role == manager                 # drop the arcane=true label: any manager will do
arcane_data_volume_driver_opts:
  type: nfs
  o: addr=nas01.example.com,nfsvers=4,rw
  device: ":/volume1/docker/arcane/data"
arcane_postgres_volume_driver_opts:
  type: nfs
  o: addr=nas01.example.com,nfsvers=4,rw
  device: ":/volume1/docker/arcane/postgres"
```

This adds a `db` service (`postgres:18-alpine`) to the `arcane` stack. Arcane connects to it as `db` over the
stack's own network, so no database port is published; the role builds the connection URL and stores it, like the
password, as a Docker secret. Both services then move to another manager if theirs fails. Create the two NFS
directories first.

Things to know before you enable it:

- **Leave the `arcane` stack alone in the UI.** Arcane lists its own stack like any other. Stopping or removing it
  there takes Arcane down with it; run the job again to restore it.
- **Enabling it starts Arcane with an empty database.** Nothing is migrated from SQLite, so settings, users and
  environments have to be set up again. Enable it on a new install if you can.
- **Start order.** Swarm ignores `depends_on`, so Arcane can start before the database is ready; it restarts
  until the database answers, usually within a minute.
- **NFS.** Mount with `hard` (the Linux default; don't add `soft`). The Postgres image changes the owner of its
  data directory on start, so the export must allow root to change ownership (no root squashing), as the
  existing stacks' Postgres volumes already need.
- **One database server only.** The service runs one replica and stops the old one before starting a new one.
  Never scale it up: two servers on the same data directory corrupt it.
- **The NAS becomes the single point of failure** for Arcane, as it already is for the NFS-backed stacks.
- **Back it up**, e.g. with a `pg_dump` job or a backup sidecar like the Semaphore stack's.

To use a PostgreSQL server outside the swarm instead, leave `arcane_postgres_enabled` off and set
`arcane_database_url` (from the credential).

Turning `arcane_postgres_enabled` off again removes the `db` service, but not its volume or data.

On a new install, sign in as `arcane` / `arcane-admin` (Arcane's own install docs still say `admin` / `admin`);
Arcane then asks for a new password.

Arcane's Swarm pages work against this manager environment directly. Arcane agents on the other nodes, which give
per-node container views, are optional. They are registered in the Arcane UI (**Environments**), which issues a
token for each node, so they aren't deployed by this repo yet.

## AWX setup

1. **Project**: this repository.
2. **Inventory**: a group `docker_swarm` with the child groups `docker_swarm_managers`, `docker_swarm_workers` and
   `docker_swarm_remove`. Add host variables such as `docker_swarm_node_labels` there.
3. **Credentials**:
   - A Machine credential for the `ansible` user (SSH key, sudo).
   - A custom credential type for Arcane, so the key never sits in extra vars:

     ```yaml
     # Input configuration
     fields:
       - {id: encryption_key, label: Encryption key, type: string, secret: true}
       - {id: postgres_password, label: Bundled PostgreSQL password, type: string, secret: true}
       - {id: database_url, label: External database URL, type: string, secret: true}
       - {id: oidc_client_secret, label: OIDC client secret, type: string, secret: true}
     required: [encryption_key]
     # Injector configuration
     extra_vars:
       arcane_encryption_key: "{{ encryption_key }}"
       arcane_postgres_password: "{{ postgres_password | default('') }}"
       arcane_database_url: "{{ database_url | default('') }}"
       arcane_oidc_client_secret: "{{ oidc_client_secret | default('') }}"
     ```

4. **Job template**: playbook `site.yml` with both credentials attached, **Privilege Escalation** on, and
   **Prompt on launch** for the limit.

The execution environment needs the `ansible.posix` collection ([requirements.yml](requirements.yml)).
Nothing needs to be installed in Python on the nodes: the roles use the `docker` CLI.

## Development and CI

The workflows and lint configs match the other `SAL9000-HomeLab` Ansible repositories:

- **Ansible CI** (`.github/workflows/ansible-ci.yml`), on pushes to `main` and every pull request:
  `yamllint`, `ansible-playbook --syntax-check site.yml`, `ansible-lint` ([.ansible-lint](.ansible-lint)).
- **Linting Validation** (`.github/workflows/ci.yml`), on pull requests: markdownlint, linkspector, yamllint.

Run the same checks locally:

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install "yamllint>=1.30" "ansible>=2.15" "ansible-lint>=6"
ansible-galaxy collection install -r requirements.yml
yamllint -f parsable .
ansible-playbook -i inventory/hosts.yml --syntax-check site.yml
ansible-lint .
npx markdownlint-cli2 "**/*.md" "#.venv"
```

Add a line under `## [Unreleased]` in [CHANGELOG.md](CHANGELOG.md) with each change. Pushing a `vX.Y.Z` tag
publishes that version's section as a GitHub release.
