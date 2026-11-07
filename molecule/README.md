<!--
SPDX-FileCopyrightText: 2018-2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (12/Bookworm or newer)
  - **Ubuntu** (22.04 or newer)
- `root` access on the computer which Molecule runs against
- [Ansible](https://www.ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default; others need it installed (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

Currently there are 3 testing scenarios available.
None of them talks to a real orchestrator or model: the orchestrator's URL points at a port nothing listens on.

### `default`

Tests a worker in `container` mode, with a customized image carrying Codex, Pi, mise, an apt package and a custom Dockerfile step.
The worker's uid is unknown to the image.
The scenario checks that:

- the container runs the customized image with its hardening in place, without a Docker socket or published ports;
- as the worker's user, and in the emptied environment that tasks get, all tools report their pinned versions;
- a project's mise configuration works, and does not get in Pi's way;
- the home, the workspaces, the credentials and `/tmp` are writable, and the rest is not;
- the token is in its file only;
- the worker keeps retrying without restarting over 45 seconds;
- `docker stop` stops it through SIGTERM;
- the sweep removes stale attempts on start, follows no symbolic links, and leaves a quarantined worker alone.

### `host`

Tests a worker in `host` mode, with a stand-in for Codex on the host.
The scenario checks that the binary extracted from the image has the pinned version and is root-owned, that it runs as the worker's user, with its additional group and no container, that it keeps retrying, stops on SIGTERM, and that the sweep removes root-owned files.

### `published`

Tests a worker in `container` mode with nothing to customize.
The scenario checks that the published image runs as is, and nothing gets built.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
molecule test --scenario-name host
molecule test --scenario-name published
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```
