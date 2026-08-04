---
title: Forgejo Runner on nested, rootless Podman
date: 2026-08-03 19:56:56 +0100
categories: [Linux]
tags: [containers, linux]
---

The official [Forgejo Runner Docker installation guide](https://forgejo.org/docs/latest/admin/actions/installation/docker/) recommends running a privileged Docker-in-Docker container for the runner's Docker host. This is well-intentioned, as it provides the runner with a clean environment to kick off jobs on. However, it requires granting [risky](https://docs.docker.com/engine/containers/run/#runtime-privilege-and-linux-capabilities) special privileges (note: the DinD container's privileged state doesn't automatically allow for container escape, because the actual less-trusted containers are running without it, but it's still unnecessary).

There's a safer way. We can run Podman, a container engine supporting rootless execution, [inside a rootless Podman container](https://www.redhat.com/en/blog/podman-inside-container), like a [matryoshka](https://en.wikipedia.org/wiki/Matryoshka_doll)! This effectively means that root privileges are never, ever required for Forgejo Actions to run.

I will be using [Quadlets](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) for this post. Regular `podman` commands will also be provided for your convenience.

## 0. Prerequisites

- Podman (v4.0+) installed.
- User namespaces enabled on the host - relevant guide [available here](https://www.baeldung.com/linux/kernel-enable-user-namespaces). Most distros as of 2026 have it on by default.

## 1. Create a network for both containers

The Podman container and Forgejo Runner require a separate, shared network to communicate between each other. Using host networking or port mappings would be highly insecure in this case, because it would grant any other app running on your server access to Runner jobs.

In Quadlet, below in `~/.config/containers/systemd/forgejo_runner.network` should be all you need:

```ini
[Network]
NetworkName=forgejo_runner
```

Alternatively, run:

```bash
podman network create forgejo_runner
```

## 2. Start the nested Podman container

Instead of using `--privileged` outright to a new Podman instance, we're providing it with just the permissions it needs to launch its own containers:

- `/dev/fuse` access to mount filesystems,
- `/dev/net/tun` access and unmasked `/proc/sys` for rootless networking,
- disabled SELinux labels (may or may not work without, but this setting is recommended by Red Hat's own blog),
- `CONTAINERS_CGROUP_MANAGER=cgroupfs` to use `cgroupfs` as the container group manager, to work around systemd issues.

No volume mounts are needed here. This container is completely disposable.

Quadlet (`~/.config/containers/systemd/forgejo_runner_podman.container`):

```ini
[Unit]
Description=Isolated Podman for forgejo-runner
After=network-online.target

[Container]
Image=quay.io/podman/stable
Exec=podman system service --time=0 tcp:0.0.0.0:8080
HostName=forgejo_runner_podman
Network=forgejo_runner.network
AddDevice=/dev/fuse
AddDevice=/dev/net/tun
Unmask=/proc/sys
Environment=CONTAINERS_CGROUP_MANAGER=cgroupfs
SecurityLabelDisable=true
User=podman
Group=podman
AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Equivalent `podman run` command:

```bash
podman run \
  --name forgejo_runner_podman \
  --network forgejo_runner \
  --hostname forgejo_runner_podman \
  --device /dev/fuse \
  --device /dev/net/tun \
  --security-opt label=disable \
  --security-opt unmask=/proc/sys \
  --env CONTAINERS_CGROUP_MANAGER=cgroupfs \
  --user podman:podman \
  --label "io.containers.autoupdate=registry" \
  --restart always \
  quay.io/podman/stable \
  podman system service --time=0 tcp:0.0.0.0:8080
```

You should see `level=warning msg="Using the Podman API service with TCP sockets without TLS is not recommended, please see podman system service manpage for details"`. This is mitigated through the use of an internal network. Add `--detach` to the arguments to run the container in the background.

## 3. Configure and run the Forgejo Runner

### Configuration

First, we need the configuration details:

1. On your Forgejo instance, navigate to Settings -> Actions -> Runners.
2. Click on the "Create new runner" button. Enter a creative name, then Create.
3. Copy the entire code block under *Using the runner configuration file*, and paste it into a new YAML file.

You also need to specify *labels* for your runner, which define what types of workloads your Runner can handle. See [the Forgejo Actions documentation](https://forgejo.org/docs/v15.0/admin/actions/configuration/) for details.

A working configuration looks something like this:

```yaml
server:
  connections:
    forgejo:
      url: https://codeberg.org/
      uuid: YOUR_UUID_HERE
      token: YOUR_TOKEN_HERE

runner:
  labels:
    - debian-latest:docker://mcr.microsoft.com/devcontainers/javascript-node:latest
```

Please note that for special Actions to run (like GitHub's [checkout](https://github.com/actions/checkout)), you need a container with Node.js included.

### Execution

This Quadlet runs the Forgejo Runner as a rootless user (uid, gid `1000`) inside the container, and instructs it to connect to our Podman instance via the `DOCKER_HOST` environment variable. In order to run it, you need to write the above configuration into a file `/data/forgejo_runner/runner-config.yml`. This file name is non-negotiable, unless you change the `--config` argument passed to the `forgejo-runner` daemon.

Note the [UserNS=keep-id](https://docs.podman.io/en/v4.6.1/markdown/options/userns.container.html) option. It's crucial whenever you're running rootless Podman containers. If it were not present, Podman would map the host user's ID to the container's `root`, meaning only this superuser would be able to read Forgejo Actions' data directory. This is undesirable, because the runner itself is running as an unprivileged user inside, and it needs access to its configuration plus a cache. `keep-id:uid=1000,gid=1000` maps the host user to the container's user and group `1000`, granting it this entry.

`~/.config/containers/systemd/forgejo_runner.container`:

```ini
[Unit]
Name=Forgejo CI runner
After=network-online.target

[Container]
Image=data.forgejo.org/forgejo/runner:12
Environment=DOCKER_HOST=tcp://forgejo_runner_podman:8080
Network=forgejo_runner.network
Exec=forgejo-runner daemon --config runner-config.yml
Volume=/data/forgejo_runner:/data:z
User=1000
Group=1000
UserNS=keep-id:uid=1000,gid=1000

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Equivalent `podman run` command:

```bash
podman run \
  --name forgejo_ci_runner \
  --network forgejo_runner \
  --env DOCKER_HOST=tcp://forgejo_runner_podman:8080 \
  --volume /data/forgejo_runner:/data:z \
  --user 1000:1000 \
  --userns keep-id:uid=1000,gid=1000 \
  --restart always \
  data.forgejo.org/forgejo/runner:12 \
  forgejo-runner daemon --config runner-config.yml
```

## 4. Profit

Manually start the Quadlets.

```bash
systemctl --user daemon-reload
# Change these according to the .container file names
systemctl --user start --now forgejo_runner_podman
systemctl --user start --now forgejo_runner
```

If no cosmic rays attacked your server, you should now see the runner you previously created in an *idle* state on the Runners page.

Using it is as simple as making a new file inside your repository, e.g.: `.forgejo/workflows/build.yaml`:

```yaml
name: Rust Build & Test

on: [push]

jobs:
  build:
    # The runs-on value must match the label set earlier in the config
    runs-on: debian-latest
    
    # Override the container image used (optional)
    container:
      image: rust:latest

    steps:
      # The official 'rust' image does not include Node.js, which is required 
      # by actions/checkout. We must install it first.
      - name: Install Node.js
        run: |
          apt-get update
          apt-get install -y nodejs

      - name: Checkout code
        uses: actions/checkout@v4

      - name: Check Rust version
        run: |
          rustc --version
          cargo --version

      - name: Build
        run: cargo build --verbose
```

Enjoy!
