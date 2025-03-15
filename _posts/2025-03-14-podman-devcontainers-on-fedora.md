---
title: Using Rootless Podman with Dev Containers on Fedora
date: 2025-03-14 19:56:56 +0100
categories: [Linux]
tags: [containers, linux]     # TAG names should always be lowercase
---

I use Visual Studio Code [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) to work on my projects in a clean environment where there's no possibility of breaking my system or polluting it with unnecessary packages.

Setting them up on Linux in a secure fashion, however, requires extra work. The official VS Code [documentation](https://code.visualstudio.com/docs/devcontainers/containers) recommends adding your user to the `docker` group, allowing all apps on your computer to access rootful Docker and its far-reaching capabilities. This is deeply irresponsible security-wise.

Podman, a Docker replacement, not only is completely rootless by default, but also supported by the Dev Containers extension. Best of both worlds!

## 1. Install podman-docker

`podman-docker` "redirects" all Docker commands to Podman. This lets us avoid extra configuration in VS Code because it's tricked into thinking that the former is installed.
```bash
sudo dnf install podman-docker
```
{: .nolineno }

This step is optional. You can proceed without it by manually changing the Docker path in the Dev Containers extension preferences.

## 2. Adjust the SELinux context of your project's directory

By default, Fedora's SELinux configuration prevents containers from accessing the home folder, resulting in `Permission denied` errors if your projects are located there. This behavior can be changed with the command below.
```sh
sudo chcon -R -t container_file_t /path/to/project
```
{: .nolineno }

Red Hat provides further context in the RHEL [documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-security-enhanced_linux-working_with_selinux-selinux_contexts_labeling_files) on SELinux contexts. While this works for me, note that I am no expert on SELinux and I haven't thoroughly researched if this causes unintended side effects. However, it's surely better than disabling SELinux and/or `chmod 777`-ing everything :)

## 3. Add additional options to your project's `devcontainer.json`

The two properties below allow the container and underlying VS Code server to run as intended. Courtesy of [Troy Kershaw](https://www.troykershaw.com/posts/vscode-dev-containers-podman/), his blog post explains them in greater detail.
```json
"runArgs": ["--userns=keep-id"],
"containerEnv": { "HOME": "/home/vscode" }
```
The correct value of the `HOME` environment variable varies per container - for instance, Node.js containers use `/home/node`, Rust and many others - `/home/vscode`. 

Now, reopen your project in the container. Have fun!

## Bonus: VS Code Flatpak setup

Using this setup with the Flatpak is straightforward.

First, create a shell script with below contents and make it executable:

```sh
#!/bin/sh
exec flatpak-spawn --host podman "$@"
```

It's a simple wrapper that lets VS Code interact with Podman running outside the Flatpak sandbox.

Allow the Flatpak to access Podman:

```sh
flatpak override \
  # Allow access to the Podman socket. Optional, may or may not work without 
  --filesystem=xdg-run/podman:ro \
  # Required for image builds
  --filesystem=/tmp \
  com.visualstudio.code
```

Finally, update the Docker Path in the Dev Containers extension settings to point to the newly created shell script.

If you run into issues, try enabling the Podman socket:

```bash
systemctl --user enable --now podman.socket
```
