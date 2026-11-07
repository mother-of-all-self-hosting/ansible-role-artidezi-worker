<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up an ArtiDezi worker

This is an [Ansible](https://www.ansible.com/) role which installs an [ArtiDezi](https://github.com/devture/artidezi) worker.

A worker polls its orchestrator over HTTPS with its own token, and runs a coding-agent harness ([Codex](https://github.com/openai/codex) or [Pi](https://github.com/earendil-works/pi)) on repository checkouts.
It only makes outgoing connections, so it needs no reverse proxy and no published ports.

[ArtiDezi's installation guide](https://github.com/devture/artidezi/blob/main/docs/user-guide/installing.md) describes how the pieces fit together.

>[!WARNING]
> A worker is untrusted and it is not a sandbox.
> It runs repository code and a coding agent, 1 task after another in the same environment, and every task can read the worker's token and model credentials.
> Treat its machine as expendable: use a dedicated one (e.g. a VM with a firewall), with nothing else on it, and enforce network isolation there.
> See [what each boundary guarantees](https://github.com/devture/artidezi/blob/main/docs/architecture/security-and-storage.md).

## Prerequisites

- An orchestrator of the same ArtiDezi release, with this worker under `workers` in a worker pool, by the id in `artidezi_worker_id` (the inventory hostname by default), and a token for it.
- Every directory above `artidezi_worker_base_path` is owned by root and writable only by root, or the role refuses to install: systemd runs a script from there as root.
  The MASH playbook's base directory (`/mash`) is.
- Docker with the Buildx plugin and BuildKit, in both modes (`host` mode needs it to get the binary).
- In `host` mode, the worker's user exists on the host, and so do `git`, the harnesses and whatever the projects' commands need.

## Adjusting the playbook configuration

To enable the worker with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# artidezi-worker                                                      #
#                                                                      #
########################################################################

artidezi_worker_enabled: true

artidezi_worker_orchestrator_url: https://artidezi.example.com
artidezi_worker_pool: maintenance
artidezi_worker_token: "{{ vault_artidezi_worker_token }}"

artidezi_worker_codex_enabled: true

########################################################################
#                                                                      #
# /artidezi-worker                                                     #
#                                                                      #
########################################################################
```

`artidezi_worker_uid` and `artidezi_worker_gid` need values too, if your playbook does not provide them.

Check [`defaults/main.yml`](../defaults/main.yml) for the other options, such as `artidezi_worker_tags` and `artidezi_worker_poll_seconds`.

## Choosing a mode

Both modes use the same published image, which bundles no harness.

### Container mode

`artidezi_worker_mode: container` is the default.
The worker runs in a container, and so does everything a task does.

Tooling comes from a customized image, built on the host `FROM` the published image, only when one of the settings below asks for something.
Otherwise, the published image runs as is.

| Setting | Puts in the image |
| --- | --- |
| `artidezi_worker_codex_enabled` | Codex, from its release archive at `artidezi_worker_codex_version` |
| `artidezi_worker_pi_enabled` | Pi, from npm at `artidezi_worker_pi_version`, with its own Node.js (`artidezi_worker_pi_node_version`) and the `fd-find` and `ripgrep` packages it needs |
| `artidezi_worker_mise_enabled` | [mise](https://mise.jdx.dev/) at `artidezi_worker_mise_version`, and `artidezi_worker_mise_packages` (by default, `just`) |
| `artidezi_worker_container_image_customizations_packages_to_install_custom` | apt packages |
| `artidezi_worker_container_image_customizations_dockerfile_body_custom` | Your own Dockerfile steps, which run as root |

Pi's Node.js stays out of the projects' way: it is not on PATH.
A project that needs Node.js gets it from mise or apt.

```yaml
artidezi_worker_codex_enabled: true
artidezi_worker_pi_enabled: true

artidezi_worker_mise_enabled: true
artidezi_worker_mise_packages:
  - just@1.59
  - node@24

artidezi_worker_container_image_customizations_packages_to_install_custom:
  - build-essential
  - curl
  - python3
```

The published image is minimal, so add what the projects' commands need.

The worker clears the environment and creates a new home directory for each attempt, so tools from `artidezi_worker_mise_packages` are linked into `/usr/local/bin` rather than used through mise's shims.
Tools from a project's own mise configuration install into that attempt's home directory when it runs `mise install` or `mise exec`, and repositories in the workspaces count as trusted.

#### What the container is hardened with

| Choice | Why |
| --- | --- |
| `--cap-drop=ALL`, `--security-opt=no-new-privileges` | The worker runs as an unprivileged user, and nothing a task needs takes capabilities or setuid programs (there is no `sudo` or `apt-get install` at task time) |
| No Docker socket | It would be root on the host. Projects that need Docker belong in `host` mode |
| `--read-only`, with a writable home, workspaces, credentials and `/tmp` | Tasks can't change the image's files. `artidezi_worker_container_read_only_enabled: false` turns this off for a tool insisting on writing elsewhere |
| `/tmp` as a tmpfs of 2 GiB, with `exec` | Test suites and build tools run things from there (`artidezi_worker_container_tmpfs_options`) |
| A network of its own, no published ports | Other containers on the host are not on it. The host itself and everything else the host can reach remain reachable: firewall those outside the worker |
| `--init` | Reaps the orphans that tasks leave behind |
| The worker's configuration and token mounted read-only | A task can't change them. It can still read the token: the worker needs it |
| A generated `/etc/passwd` and `/etc/group` | `ssh`, `whoami` and Node.js's `os.userInfo()` fail for a user without a name |

The default seccomp and AppArmor profiles stay in place.
There are no memory, CPU or process limits by default, as what a project's tests need differs too much: add them with `artidezi_worker_container_extra_arguments_custom` (e.g. `--memory=8g`, `--pids-limit=4096`).

What this does not do: isolate tasks from each other.
The home directory persists, and the worker's token and the model credentials are within reach of every task.
Harness sandboxing is off too: ArtiDezi runs Codex with full access, as the environment around it is the boundary.

### Host mode

`artidezi_worker_mode: host` is an escape hatch for workers that need more privilege, such as projects needing Docker.
The role pulls the image, extracts its static binary to `artidezi_worker_binary_path` (root-owned, like everything else under `artidezi_worker_bin_path`), and runs it under systemd as `artidezi_worker_uid`.

The worker and its tasks then use the host's tools, so nothing is confined except by that user's permissions, and the role installs no harness or tooling.
`artidezi_worker_codex_enabled` and `artidezi_worker_pi_enabled` only make the worker offer `codex` and `pi` from PATH, and the other customization settings do nothing.
Set `artidezi_worker_codex_binary` or `artidezi_worker_pi_binary` for an absolute path.

Don't share this user with anything else.
In the MASH playbook, all services run as the same user by default, which in this mode gives every task access to their data: give the worker a machine of its own, or a user of its own. A user of its own can't enter the playbook's base directory (`/mash`), so also set `artidezi_worker_base_path` outside of it (e.g. `/opt/artidezi-worker`).

To let tasks use Docker:

```yaml
artidezi_worker_mode: host
artidezi_worker_systemd_supplementary_groups: [docker]
artidezi_worker_tags: [docker-support]
```

>[!WARNING]
> Membership of the `docker` group makes every task root on the host.

## Providing harness credentials

Harness credentials live in `<credentials_root>/<credential_ref>/`, owned by the worker's user, where `credential_ref` comes from the agent profile in the orchestrator's configuration.
`credentials_root` is `artidezi_worker_credentials_path` on the host (`/artidezi-worker/data/credentials` by default, and in the MASH playbook usually `/mash/artidezi-worker/data/credentials`).

| Harness | `auth_mode` | File |
| --- | --- | --- |
| Codex | `openai_api_key` | `api_key` |
| Codex | `chatgpt_login` | `auth.json` |
| Pi | `openai_api_key` | `api_key` |

The role does not manage these files, and uninstalling the role removes them.

### API keys

On the host, as root:

```sh
credentials=/artidezi-worker/data/credentials # artidezi_worker_credentials_path
install -d -o WORKER_UID -g WORKER_GID -m 0700 "$credentials/openai"
install -o WORKER_UID -g WORKER_GID -m 0600 api_key "$credentials/openai/api_key"
```

### ChatGPT login for Codex

`codex login` is interactive, so the role can't do it.
Sign in on each worker separately, once the worker is running.

In `container` mode:

```sh
docker exec artidezi-worker mkdir -p /var/lib/artidezi-worker/credentials/CREDENTIAL_REF
docker exec -it --env CODEX_HOME=/var/lib/artidezi-worker/credentials/CREDENTIAL_REF artidezi-worker codex login --device-auth
```

In `host` mode, as root:

```sh
credential=/artidezi-worker/data/credentials/CREDENTIAL_REF
install -d -o WORKER_UID -g WORKER_GID -m 0700 "$credential"
setpriv --reuid=WORKER_UID --regid=WORKER_GID --clear-groups env HOME="$credential" CODEX_HOME="$credential" codex login --device-auth
```

Approve the printed code in a browser on another machine.
Codex writes `auth.json` into that directory, and the worker takes it from there, copying it into each attempt.

>[!WARNING]
> Don't copy `auth.json` between workers: they would share a single-use refresh token, and a rotation by one would log the others out.

To replace a working login, sign in with `CODEX_HOME` pointing to a temporary directory on the same filesystem, and move `auth.json` into place once it worked, so that a cancelled login does not leave the worker logged out.
Do it with the worker idle: it refuses to save a rotated login over a file that changed during an attempt.

## Installing

After configuring the playbook, run the installation command of your playbook:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

The shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`.

The worker appears in the orchestrator once it reaches it, and its harnesses need to match the `harness_version_constraint` of the agent profiles, or jobs wait in the queue.
Leave it at `'*'`, unless you have a reason not to: an exact version leaves the worker's jobs waiting after every update of `artidezi_worker_codex_version` or `artidezi_worker_pi_version`.

## Upgrading

Upgrade the orchestrator and its workers to the same release.

(Re)starting the service stops the attempt the worker is running, so run the playbook when the worker is idle.

## Stopping and interrupted attempts

On SIGTERM (`systemctl stop`, `docker stop`), the worker stops its attempt and exits.
`artidezi_worker_stop_grace_time_seconds` (120 by default) bounds that, and systemd's timeouts follow from it.

The worker never resumes an attempt.
Before each start, `bin/sweep-workspaces` runs as root and removes attempt workspaces that are left over from a reboot, a crash or a failed cleanup, including root-owned files.

A worker that could not stop a process puts `worker-quarantine.json` into its workspaces directory.
It then takes no work, and the sweep keeps everything as evidence.
Look into what happened, remove the marker and restart the service.

## Troubleshooting

The worker logs to the systemd journal:

```sh
journalctl -fu artidezi-worker
```

While it can't reach the orchestrator, or a harness can't be run, it logs `worker iteration failed` and retries every `artidezi_worker_poll_seconds` seconds.
