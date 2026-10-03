# ans-docker

Builds a highly available Docker Swarm on Rocky Linux 10 hosts and deploys [Arcane](https://getarcane.app/docs)
to manage it. Run `site.yml` from AWX. Each run brings the cluster in line with the inventory, so adding, removing,
promoting or demoting a node only takes an inventory change and another run.

| Role | What it does |
| --- | --- |
| [docker_engine](roles/docker_engine) | Installs Docker CE from Docker's EL repo, writes `daemon.json`, opens the swarm ports in firewalld, installs `nfs-utils` for NFS volumes |
| [docker_swarm](roles/docker_swarm) | Creates the swarm, joins/promotes/demotes/drains/removes nodes, applies node labels, creates shared overlay networks |
| [keepalived](roles/keepalived) | Shares floating IPs across the managers; an IP can follow a service published on host-mode ports |
| [arcane](roles/arcane) | Runs Arcane as a swarm service on a manager, optionally with its own PostgreSQL database, with its secrets stored as Docker secrets |

The values committed in `inventory/` and `group_vars/` are placeholders (`example.com`). Put your real hosts in the
AWX inventory and your secrets in AWX credentials, or in local files that are gitignored (`local/`,
`extra-vars*.yml`). Never commit credentials: this repository is public.

Tested on Rocky Linux 10 with Docker CE 29.8, Arcane v2.13.1 and ansible-core 2.16 (AWX): creating the swarm,
re-running unchanged, promoting workers to managers, removing and re-adding a node, and Arcane on SQLite and on the
bundled PostgreSQL over NFS.

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
4. Delete the host from the inventory, or take it out of `docker_swarm_remove` and run the job again to rejoin it.

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

### keepalived

| Variable | Default | Notes |
| --- | --- | --- |
| `keepalived_instances` | `[]` | Floating IPs. Each: `name`, `virtual_ip` (with prefix, e.g. `192.0.2.50/24`), `virtual_router_id` (1–255, unique on the network), optional `track_tcp_port`. Empty stops keepalived. |
| `keepalived_interface` / `keepalived_address` | the default IPv4 interface and address | Per host. Used for VRRP and for the port check. |
| `keepalived_priority` | `100` | Per host. |
| `keepalived_check_interval` / `_fall` / `_rise` | `2` / `2` / `2` | Port check timing; failover takes about interval × fall seconds. |
| `keepalived_firewalld_manage` / `_zone` | `true` / `public` | Allows the VRRP protocol when firewalld is running. |

See [Floating IPs](#floating-ips) below.

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
| `arcane_version` | `v2.14.0` | Image tag of `ghcr.io/getarcaneapp/manager`. |
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

Put these on the `docker_swarm` group in the AWX inventory (or in the job template's variables) as top-level
variables, not nested under another key.

This adds a `db` service (`postgres:18-alpine`) to the `arcane` stack. Arcane connects to it as `db` over the
stack's own network, so no database port is published; the role builds the connection URL and stores it, like the
password, as a Docker secret. Both services then move to another manager if theirs fails.

Before the first run:

1. Create the two NFS directories, exported to every manager.
2. Check that each manager can mount them and that root can change ownership there (Postgres and Arcane both change
   the owner of their data directory on start). From AWX (**Inventories → Run Command**, module `shell`, privilege
   escalation on) or the CLI:

   ```sh
   ansible docker_swarm_managers -b -m shell -a '
     set -e
     d=$(mktemp -d)
     mount -t nfs4 nas01.example.com:/volume1/docker/arcane/postgres "$d"
     trap "umount $d; rmdir $d" EXIT
     f="$d/probe-$(hostname -s)"
     touch "$f"; chown 70:70 "$f"; ls -ln "$f"; rm "$f"'
   ```

   "Operation not permitted" means root is squashed; on Synology, set the NFS rule's **Squash** to "No mapping".
3. If Arcane already runs with a local data volume, remove it first. Docker reuses an existing volume of the same
   name, so the node that ran Arcane would otherwise keep using its local copy instead of NFS:

   ```sh
   sudo docker stack rm arcane      # on a manager
   sudo docker volume rm arcane_data  # on the node that ran Arcane, once the stack is gone
   ```

Things to know before you enable it:

- **Leave the `arcane` stack alone in the UI.** Arcane lists its own stack like any other. Stopping or removing it
  there takes Arcane down with it; run the job again to restore it. The job deploys the stack with `--prune`, so
  anything added to it by hand is removed on the next run.
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

#### Single sign-on (OpenID Connect)

Arcane can sign users in through an OpenID Connect provider such as authentik, and give them roles from their
groups. Everything is set with variables, so it survives redeploys: Arcane's environment settings override what's
saved in its UI. With authentik, for example:

1. In authentik, create an **OAuth2/OpenID Provider** and an **Application** with the slug `arcane`:
   - **Redirect URI** (Strict): `https://arcane.example.com/auth/oidc/callback`, the public `arcane_app_url`
     followed by `/auth/oidc/callback`.
   - **Client ID:** authentik generates a random one. Change it to `arcane`, or use the generated one as
     `OIDC_CLIENT_ID` below.
   - Keep the default scopes (`openid`, `email`, `profile`). `profile` includes the user's `groups`, which the role
     mappings below use.
2. Put the provider's **Client Secret** in the AWX credential's `oidc_client_secret` field (see
   [AWX setup](#awx-setup)). The role stores it as a Docker secret, and Arcane reads it as
   `OIDC_CLIENT_SECRET_FILE`.
3. Set these on the inventory's `docker_swarm` group:

   ```yaml
   arcane_app_url: "https://arcane.example.com"   # the public URL, behind the reverse proxy
   arcane_environment:
     OIDC_ENABLED: true
     OIDC_ISSUER_URL: https://auth.example.com/application/o/arcane/
     OIDC_CLIENT_ID: arcane
     OIDC_PROVIDER_NAME: authentik
     # Group name (from the groups claim) -> Arcane role. Users in no mapped group can sign in but have no
     # role until an admin gives them one. Built-in roles: role_admin, role_editor, role_no_shell_editor,
     # role_deployer, role_monitor, role_viewer.
     OIDC_ROLE_MAPPINGS: '[{"claimValue": "ServerAdmins", "roleId": "role_admin"}]'
   ```

4. Run the job template. Arcane's login page then shows **Sign in with authentik**.

> **Arcane v2.14.0 ignores the client secret file.** Its settings only read a plain `OIDC_CLIENT_SECRET`
> variable, not `OIDC_CLIENT_SECRET_FILE`, so it sends no secret (or the one saved in its database) and
> authentik answers `invalid_client` ("Client authentication failed"). This is fixed upstream
> ([getarcaneapp/arcane#4203](https://github.com/getarcaneapp/arcane/pull/4203), merged after v2.14.0). Until a
> release with the fix, save the client secret in Arcane's database once, through its API. The settings page
> can't do it: it locks every OIDC field, the secret too, when `OIDC_ENABLED` comes from the environment.
>
> 1. In Arcane, signed in as an admin, create an API key under **Settings → API Keys**.
> 2. On a manager (bash), enter both values at the prompts, so they stay out of the screen and the shell
>    history:
>
>    ```sh
>    read -rsp 'Arcane API key: ' KEY; echo
>    read -rsp 'Client secret: ' SECRET; echo
>    curl -sS -X PUT http://localhost:3552/api/environments/0/settings \
>      -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
>      -d "{\"oidcClientSecret\": \"$SECRET\"}" | head -c 300; echo
>    unset KEY SECRET
>    ```
>
>    A reply starting with `{"success":true` means it's saved. Environment `0` is the manager's own; Arcane
>    accepts authentication settings only there.
> 3. Sign in with authentik, then delete the API key.
>
> Use the same value as in the credential: after upgrading to a release with the fix, the secret from the
> credential takes over. A changed secret needs the API call again until then.

Keep the local `arcane` account as a way in when the provider can't be reached. A failed sign-in is logged
with its reason: `docker service logs --since 5m arcane_arcane 2>&1 | grep -i oidc`. `invalid_client` means
the client secret Arcane sends isn't the provider's (see the note above). The redirect URI depends on
`arcane_app_url`, so use the same public URL for both. If authentik shows "The client identifier (client_id) is
missing or invalid", `OIDC_CLIENT_ID` doesn't match the provider's Client ID.

## Floating IPs

Swarm services that publish ports in `mode: host` (like Nginx Proxy Manager, so it sees real client addresses)
answer only on the node they run on. The `keepalived` role gives them one address that moves with them:

```yaml
keepalived_instances:
  - name: proxy
    virtual_ip: 192.0.2.50/24    # a free address on the managers' network; point DNS and port-forwards here
    virtual_router_id: 51
    track_tcp_port: 443           # the IP goes to whichever manager answers on this port
```

Each manager checks every 2 seconds whether something accepts connections on its own address and that port. The
one where the service runs holds the IP; the others stand by. When the service moves to another node, the IP
follows within a few seconds. Without `track_tcp_port`, the IP stays on any running manager, which suits services
published through the ingress routing mesh (reachable on every node).

- **Run the service on managers only** (`node.role == manager`), because only managers take part.
- **VRRP runs unicast** between the managers' own addresses, and adverts from any other host are ignored. There's
  no password to manage (VRRPv2 sends it in the clear anyway).
- **The virtual IP must be free** and on the managers' subnet. On Proxmox, allow VRRP (IP protocol 112) if the VM
  firewall is on.
- **Removing a manager** (`docker_swarm_remove`) stops keepalived on it, and the other managers drop it as a peer.

Check which node holds an IP with `ip -4 addr show | grep <virtual ip>` on each manager, or
`journalctl -u keepalived` for state changes (`MASTER`, `BACKUP`, `FAULT`).

## AWX setup

1. **Project**: this repository.
2. **Inventory**: a group `docker_swarm` with the child groups `docker_swarm_managers`, `docker_swarm_workers` and
   `docker_swarm_remove`, all three created even if `docker_swarm_remove` is empty. The inventory's own name isn't a
   group, so the parent group must be called `docker_swarm` for [group_vars/docker_swarm.yml](group_vars/docker_swarm.yml)
   to apply. Put your real settings (e.g. `arcane_app_url`) on that group as top-level variables, and host
   variables such as `docker_swarm_node_labels` on the hosts.
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

## Troubleshooting

- **The Arcane deploy fails** after waiting (`arcane_deploy_wait_retries` × `arcane_deploy_wait_delay`): the error
  lists each service's replica count, recent task errors and last log lines. On a manager, the same comes from:

  ```sh
  sudo docker service ls --filter label=com.docker.stack.namespace=arcane
  sudo docker service ps --no-trunc arcane_arcane      # or arcane_db
  sudo docker service logs --tail 50 arcane_arcane
  ```

  A task that keeps restarting with `non-zero exit` is crashing; the reason is in `docker service logs`, not in
  `/var/log/messages`.
- **A floating IP isn't answering:** on each manager, `journalctl -u keepalived -n 20`. `FAULT` on every node
  means the tracked port answers nowhere: check the service is running (`docker service ps`).
- **Swarm state:** `sudo docker node ls` on any manager shows every node's status, availability and manager role.
- **Can't log in to a new Arcane:** the default account is `arcane` / `arcane-admin`, not `admin` / `admin`.

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
