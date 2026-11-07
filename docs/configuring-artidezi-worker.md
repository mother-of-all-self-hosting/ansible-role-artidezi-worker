<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up an ArtiDezi worker

This is an [Ansible](https://www.ansible.com/) role which installs a worker of [ArtiDezi](https://github.com/devture/artidezi) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service, or as a plain binary under systemd.

ArtiDezi is a self-hosted maintenance agent that pushes the boring buttons: it lets coding agents do routine maintenance on your repositories, without giving them your forge credentials.

A worker polls its orchestrator over HTTPS with its own token, and runs a coding-agent harness ([Codex](https://github.com/openai/codex) or [Pi](https://github.com/earendil-works/pi)) on repository checkouts. It only makes outgoing connections, so it needs no reverse proxy and no published ports. The orchestrator is installed separately, e.g. with [ansible-role-artidezi-orchestrator](https://github.com/mother-of-all-self-hosting/ansible-role-artidezi-orchestrator).

See the project's [documentation](https://github.com/devture/artidezi/blob/main/docs/README.md) to learn what ArtiDezi does and why it might be useful to you.

>[!WARNING]
> A worker is untrusted, and it is not a sandbox.
> It runs repository code and a coding agent, 1 task after another in the same environment, and every task can read the worker's token and the harness credentials stored on it.
> Treat its machine as expendable: use a dedicated one (e.g. a VM with a firewall), with nothing else on it, and enforce network isolation there.
> See [Security and storage](https://github.com/devture/artidezi/blob/main/docs/architecture/security-and-storage.md) in ArtiDezi's documentation for what each boundary guarantees.

## Prerequisites

- An orchestrator of the same ArtiDezi release, which lists this worker under `workers` in one of its worker pools, by the id in `artidezi_worker_id` (the inventory hostname by default), with a token for it.
- Docker, in both modes. In `container` mode, also its Buildx plugin and BuildKit, which build the customized image. In `host` mode, the role still needs Docker to get the binary out of the image.
- Every directory above `artidezi_worker_base_path` is owned by root and writable only by root, or the role refuses to install: systemd runs a script from there as root before each start. The MASH playbook's base directory (`/mash`) is.
- In `host` mode, the worker's user exists on the host, and so do `git`, the harnesses and whatever the projects' commands need.

## Adjusting the playbook configuration

To enable an ArtiDezi worker with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# artidezi_worker                                                      #
#                                                                      #
########################################################################

artidezi_worker_enabled: true

artidezi_worker_orchestrator_url: https://artidezi.example.com
artidezi_worker_pool: maintenance
artidezi_worker_id: worker-1
artidezi_worker_token: WORKER_TOKEN_HERE

########################################################################
#                                                                      #
# /artidezi_worker                                                     #
#                                                                      #
########################################################################
```

`artidezi_worker_pool` and `artidezi_worker_id` are the pool and the worker's name under it in the orchestrator's configuration, and `artidezi_worker_token` is the value of the token the orchestrator lists for this worker.

`artidezi_worker_uid` and `artidezi_worker_gid` need values too, if your playbook does not provide them. In `container` mode, the user does not need to exist on the host.

### Choose a harness

A worker needs at least 1 harness, or the role refuses to install.

To offer Codex, add the following configuration to your `vars.yml` file:

```yaml
artidezi_worker_codex_enabled: true
```

To offer Pi:

```yaml
artidezi_worker_pi_enabled: true
```

In `container` mode (the default), this installs the harness into the worker's image. In `host` mode, install it yourself: the worker uses `codex` or `pi` from the host's `PATH`, or `artidezi_worker_codex_binary` and `artidezi_worker_pi_binary`.

### Adding project tooling (optional)

In `container` mode, the role builds a customized image on the host with what the projects' commands need.

For tools that [mise](https://mise.jdx.dev/) can install, add the following configuration to your `vars.yml` file:

```yaml
artidezi_worker_mise_enabled: true
artidezi_worker_mise_packages:
  - just@1.59
  - node@24
```

These tools are linked into `/usr/local/bin`, so a plain `just` ignores the pins in a repository's `mise.toml`. A project which needs its own versions runs its commands through `mise exec`.

For apt packages:

```yaml
artidezi_worker_container_image_customizations_packages_to_install_custom:
  - build-essential
  - curl
  - python3
```

For anything else, add your own Dockerfile steps, which run as root after the role's own:

```yaml
artidezi_worker_container_image_customizations_dockerfile_body_custom: |
  RUN echo 'This is a custom step for building the customized worker image.'
```

Keep secrets out of them: they end up in the image's history.

### Setting tags (optional)

Tags route the right work to a worker. They are not a trust boundary. To set them, add the following configuration to your `vars.yml` file:

```yaml
artidezi_worker_tags: [docker-support]
```

See [Worker tags](https://github.com/devture/artidezi/blob/main/docs/user-guide/projects.md#worker-tags) in ArtiDezi's documentation for how projects use them.

### Running on the host instead of in a container (optional)

`artidezi_worker_mode: host` is an escape hatch for workers that need more privilege, such as projects needing Docker. The role extracts the image's binary and runs it under systemd, with the host's tools. Nothing confines a task except that user's permissions.

Give the worker a user of its own: sharing `mash_playbook_uid` would give every task access to other services' data. Such a user cannot enter the playbook's base directory (`/mash`), so set `artidezi_worker_base_path` outside of it too.

To let tasks use Docker, add the following configuration to your `vars.yml` file:

```yaml
artidezi_worker_mode: host
artidezi_worker_base_path: /opt/artidezi-worker
artidezi_worker_uid: WORKER_UID_HERE
artidezi_worker_gid: WORKER_GID_HERE
artidezi_worker_systemd_supplementary_groups: [docker]
artidezi_worker_tags: [docker-support]
```

>[!WARNING]
> Membership of the `docker` group makes every task root on the host.

### Adjusting the container (optional)

In `container` mode, the container runs unprivileged, with a read-only root filesystem, on a network of its own, and with no Docker socket. It does not isolate tasks from each other, or from what the host can reach: firewall that outside the worker.

- `artidezi_worker_container_read_only_enabled: false` turns the read-only root filesystem off, for a tool which insists on writing elsewhere.
- `artidezi_worker_container_extra_arguments_custom` adds limits, e.g. `[--memory=8g, --pids-limit=4096]`. There are none by default.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. Among them:
  - `artidezi_worker_poll_seconds`: how often the worker asks for work
  - `artidezi_worker_stop_grace_time_seconds`: how long the worker gets to stop its attempt, before it is killed

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, the worker connects to the orchestrator and appears on its Workers page. Before it can run a job, it needs the credentials of the model endpoint.

Leave `harness_version_constraint` in the orchestrator's agent profiles at `'*'`, unless you have a reason not to: with an exact version, jobs wait after every harness update.

### Providing the harness credentials

Harness credentials live in `<credentials_root>/<credential_ref>/`, owned by the worker's user, where `credential_ref` comes from the model endpoint in the orchestrator's configuration. On the host, `credentials_root` is `artidezi_worker_credentials_path` (`/artidezi-worker/data/credentials` by default, and in the MASH playbook `/mash/artidezi-worker/data/credentials`).

| Harness | `auth_mode` | File |
| --- | --- | --- |
| Codex | `openai_api_key` | `api_key` |
| Codex | `chatgpt_login` | `auth.json` |
| Pi | `openai_api_key` | `api_key` |

The role does not manage these files.

#### API keys

On the server, as root, with the worker's uid and gid:

```sh
credentials=/mash/artidezi-worker/data/credentials # artidezi_worker_credentials_path
install -d -o WORKER_UID -g WORKER_GID -m 0700 "$credentials/openai"
install -o WORKER_UID -g WORKER_GID -m 0600 api_key "$credentials/openai/api_key"
```

#### ChatGPT login for Codex

This needs `auth_mode: chatgpt_login` on the model endpoint in the orchestrator's configuration. Sign in on each worker separately, once it is running.

In `container` mode, adjusting the container's name to `artidezi_worker_identifier` (e.g. `mash-artidezi-worker`):

```sh
docker exec artidezi-worker mkdir -p /var/lib/artidezi-worker/credentials/CREDENTIAL_REF
docker exec -it --env CODEX_HOME=/var/lib/artidezi-worker/credentials/CREDENTIAL_REF artidezi-worker codex login --device-auth
```

In `host` mode, as root:

```sh
credential=/opt/artidezi-worker/data/credentials/CREDENTIAL_REF # under artidezi_worker_credentials_path
install -d -o WORKER_UID -g WORKER_GID -m 0700 "$credential"
setpriv --reuid=WORKER_UID --regid=WORKER_GID --clear-groups env HOME="$credential" CODEX_HOME="$credential" codex login --device-auth
```

Approve the printed code in a browser on another machine.

>[!WARNING]
> Do not copy `auth.json` between workers: they would share a single-use refresh token, and a rotation by one would log the others out.

### Upgrading and restarting

Upgrade the orchestrator and its workers to the same release.

The role restarts the worker when its configuration, token, image or binary changed. Restarting a worker (also with `systemctl restart`) stops the attempt it is running, and the worker never resumes an attempt. The orchestrator reruns an interrupted attempt only when its pool sets `lost_attempt_retries`, which is 0 by default in ArtiDezi, so either run the playbook while the worker is idle, or set that in the orchestrator's configuration. See [Lost execution recovery and multiple workers](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#lost-execution-recovery-and-multiple-workers) in ArtiDezi's documentation.

Before each start, `bin/sweep-workspaces` under `artidezi_worker_bin_path` runs as root and removes the attempt workspaces left over from a reboot, a crash or a failed cleanup, including root-owned files.

### Uninstalling

>[!WARNING]
> Uninstalling (`artidezi_worker_enabled: false`) deletes `artidezi_worker_base_path`, and with it the harness credentials. A ChatGPT login has to be enrolled again on a reinstalled worker.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu artidezi-worker` (or how you/your playbook named the service, e.g. `mash-artidezi-worker`).

While the worker cannot reach the orchestrator, or a harness cannot be run, it logs `worker iteration failed` and retries every `artidezi_worker_poll_seconds` seconds.

### The worker takes no work

A worker that could not stop a task's process puts a `worker-quarantine.json` marker into its workspaces directory (`artidezi_worker_workspaces_path`). It then takes no work, and the sweep before each start keeps everything in place as evidence. Look into what happened, remove the marker and restart the service. See [Investigate stuck work](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#investigate-stuck-work) in ArtiDezi's documentation.

A worker also waits when no agent profile matches its harnesses and their versions. Check `harness_version_constraint` in the orchestrator's configuration.
