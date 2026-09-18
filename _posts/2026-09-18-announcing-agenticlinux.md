---
title: "Announcing AgenticLinux: an immutable desktop for agents"
date: 2026-09-18 01:00:00 +0100
author: Eric Curtin
description: "AgenticLinux is a bootc desktop built for working with agents: Docker Engine, llmman, Docker Sandboxes, Claude Code, Codex, OpenCode and OpenClaw preinstalled on an immutable, composefs-backed image that updates straight from Docker Hub."
image: https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/openclaw-web.webp
---

<div class="hero">
  <img src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/agenticlinux-logo-256.png" alt="AgenticLinux logo">
  <p><strong>AgenticLinux</strong> is a <a href="https://bootc-dev.github.io/bootc/">bootc</a> desktop for working with agents. Docker Engine, Docker Sandboxes, llmman and the <code>claude</code>, <code>codex</code>, <code>opencode</code> and <code>openclaw</code> agents come preinstalled on an immutable image that updates atomically from Docker Hub. Built from Fedora 44's packages, for x86_64 and aarch64, in seven desktop flavours.</p>
</div>

<figure>
  <img src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/openclaw-web.webp" alt="OpenClaw's web Control UI in Firefox on the AgenticLinux KDE desktop, reporting a health check of the machine">
  <figcaption>OpenClaw's Control UI on <code>agenticlinux:kde</code>, after being asked to check the machine's health.</figcaption>
</figure>

```
sudo bootc switch docker.io/ericcurtin044/agenticlinux:kde
sudo reboot
```

That is the whole install if you already run a bootc or ostree-based system. Everyone else can grab an [ISO from the releases page](https://github.com/ericcurtin/agenticlinux/releases). The source is on [GitHub](https://github.com/ericcurtin/agenticlinux) and the images are on [Docker Hub](https://hub.docker.com/r/ericcurtin044/agenticlinux).

## Why another agentic distro?

[Omarchy](https://omarchy.org) deserves a lot of credit. It made the case, loudly and successfully, that the operating system itself is a surface agents should be able to work on. Its pitch is *malleability*: an agent can read the config, edit the config, install the package, restart the service and inspect the logs. It is a genuinely good desktop and it has brought a lot of people to Linux.

I want the same thing agents-first, but I have come to the opposite conclusion about *where* that malleability should live.

An agent running `sudo pacman -S` or editing `/etc` on a rolling-release install is powerful, and it is also a system administrator with no change control. When the agent gets it wrong (and the ones I run get it wrong regularly) the blast radius is the whole machine. There is no "the image that booted yesterday", there is only whatever state the last dozen commands left behind. For a hobbyist desktop that is fine. For the machine I do my job on, I want something else:

- **The OS is a build artifact, not accumulated state.** Everything under `/usr` comes from one container image with one digest. If the agent wants a package in the OS, the change is a line in a `Dockerfile` that CI builds and I can review, not a mutation on my laptop.
- **Every update is atomic and has an undo.** `bootc upgrade` stages a new image; `bootc rollback` puts the old one back. A bad agent session cannot leave the OS half-updated.
- **Agents mutate things inside containers and sandboxes, not the host.** That is what Docker Engine and Docker Sandboxes are for. The host stays boring.
- **It should be reproducible across my machines.** Same image on the x86_64 workstation and the aarch64 laptop and the VM. Not "run the install script again and hope".

AgenticLinux is not a fork of anything and it is not trying to replace Omarchy's Hyprland-and-Quickshell aesthetic. It is a different answer to the same question: what should an operating system look like when an agent is going to be operating it?

## Immutable by construction

Boot AgenticLinux and look at the root filesystem:

<figure>
  <img src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/composefs.webp" alt="findmnt shows / is a read-only composefs overlay; touch /usr/bin/hello fails with Read-only file system; bootc status shows the booted image">
  <figcaption><code>/</code> is a read-only composefs mount. <code>/usr</code> cannot be written, even by root. <code>bootc status</code> shows exactly which image and digest is running.</figcaption>
</figure>

Three things are going on here.

**composefs.** The root is not a filesystem on a partition, it is a [composefs](https://github.com/containers/composefs) image: a metadata tree assembled at boot from content-addressed objects in the ostree repository, mounted read-only through overlayfs. Files are shared by content hash between deployments, so keeping the previous image around for rollback costs almost nothing, and because every object is content-addressed the whole tree can be integrity-protected with fs-verity. `/usr/lib/ostree/prepare-root.conf` has `composefs enabled = yes` and `sysroot readonly = true` baked in.

**Read-only `/usr`.** An agent that runs `curl | sudo sh` cannot quietly drop a binary into `/usr/bin`. Persistent local state lives in `/var` and `/etc` (which is a writable, three-way-merged overlay on top of the image's defaults). Home is `/var/home`. That is the whole mutable surface, and it is the surface backups and sandboxes need to care about.

**bootc.** [bootc](https://bootc-dev.github.io/bootc/) is what turns an OCI image into a bootable system and keeps it updated. `bootc status` tells you the image, the digest and the version you are running, whether an update is staged, and what you can roll back to.

## Updates come from Docker Hub

The images are plain OCI images. The whole distro is a [19-line `Dockerfile`](https://github.com/ericcurtin/agenticlinux/blob/main/Dockerfile) and a build script on top of Fedora's `fedora-ostree-desktops` bases, built weekly by GitHub Actions for both architectures and pushed to [`docker.io/ericcurtin044/agenticlinux`](https://hub.docker.com/r/ericcurtin044/agenticlinux):

| Tag      | Desktop     |
|----------|-------------|
| `kde`    | KDE Plasma  |
| `gnome`  | GNOME       |
| `sway`   | Sway        |
| `cosmic` | COSMIC      |
| `xfce`   | Xfce        |
| `budgie` | Budgie      |
| `base`   | no desktop  |

Each is also tagged `<variant>-<release>` (for example `kde-44.20260917.25`) so you can pin.

Because the artifact is a container image, the whole container toolchain applies to the operating system. You can `docker pull` it and `docker run --rm -it docker.io/ericcurtin044/agenticlinux:kde bash` to poke around a release before you boot it. You can `FROM` it and add your own layer. You can mirror it to a private registry and `bootc switch` to that. Updating the machine is:

```
sudo bootc upgrade      # fetch and stage the new image
sudo reboot             # boot into it
sudo bootc rollback     # if you don't like it
```

## Docker Engine, not a substitute

AgenticLinux ships the real thing: `docker-ce`, `containerd`, `docker-buildx-plugin` and `docker-compose-plugin` from Docker's own Fedora repository, with `docker.service` enabled out of the box. Add yourself to the `docker` group and every agent on the box can build images, run containers and use Compose exactly the way their upstream docs describe.

This matters more than it sounds for agents. Most of the "please install X so I can test this" requests an agent makes are better answered with a container than with a package on the host. On an immutable host they *have* to be, and Docker is the tool the agents already know.

The NVIDIA Container Toolkit is preconfigured as a Docker runtime, so `docker run --gpus all` works where there is a driver.

## The agents

Claude Code, Codex, OpenCode and OpenClaw are installed system-wide in the image, on an upstream Node 24 LTS (Fedora's Node links against a system SQLite that OpenClaw refuses; upstream Node bundles its own). llmman is a static binary. At the time of writing that is:

| Tool        | Version              |
|-------------|----------------------|
| `claude`    | Claude Code 2.1.275  |
| `codex`     | Codex CLI 0.155.0    |
| `opencode`  | OpenCode 1.18.31     |
| `openclaw`  | OpenClaw 2026.9.4    |
| `llmman`    | llmman 0.1.424       |
| `docker`    | Docker Engine 29.8.1 |
| `bootc`     | bootc 1.16.10        |

None of them need any setup that differs from upstream. Sign in, or point them at a local model with llmman.

## OpenClaw

[OpenClaw](https://openclaw.ai) is the always-on agent in the set: a local gateway with a web Control UI, channel integrations and a systemd user service, that can run commands on the machine it lives on. It is the agent you hand the operating system to. Here it is in Firefox on the KDE image, doing the kind of work an OS agent should do: a health check, turning on automatic OS updates, and cleaning up Docker.

<figure>
  <video controls muted playsinline preload="metadata" poster="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/openclaw-web.webp">
    <source src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/openclaw-web.webm" type="video/webm">
    <source src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/openclaw-web.mp4" type="video/mp4">
  </video>
  <figcaption>OpenClaw's Control UI on AgenticLinux KDE. Three requests: check the machine's health, find and enable bootc's update timer, reclaim Docker disk.</figcaption>
</figure>

The second request is the interesting one. bootc ships `bootc-fetch-apply-updates.timer`, disabled by default. Asked to make the OS update itself, the agent finds the unit, enables it and reads the schedule back out of it, and from then on the machine fetches and applies new images from Docker Hub on its own, with rollback if a new one fails to boot.

<figure>
  <img src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/openclaw-web-timer.webp" alt="OpenClaw enabling bootc-fetch-apply-updates.timer and reporting its next run">
  <figcaption>Automatic OS updates, switched on by the agent.</figcaption>
</figure>

Setup was the documented non-interactive onboarding, which installs the gateway as a systemd user service, then `openclaw dashboard` to open the Control UI:

```
openclaw onboard --non-interactive --accept-risk --mode local \
  --auth-choice apiKey --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-bind loopback --install-daemon --daemon-runtime node
openclaw dashboard
```

Any provider OpenClaw supports works the same way, including a local model through llmman. The agent's shell is a normal user shell on an immutable host: it can enable a timer, prune Docker or `docker run` anything it likes, and it cannot touch `/usr`.

## llmman: local models for every agent

[llmman](https://github.com/llmmanorg/llmman) is the piece that ties the local-model story together. Models are OCI images (`docker.io/ai/qwen3.8`, `hf.co/unsloth/...`), pulled with the same registry machinery as everything else on this OS. `llmman launch <agent> --model <model>` starts an inference server with the right llama.cpp build for the GPU it finds (CUDA, ROCm or Vulkan, and the image ships all three runtimes), loads the model, writes the agent's provider configuration and execs the agent against it. The same command works with a hosted provider via `--provider`.

Here are three of the agents on the image, each launched on Qwen3.8 27B (`IQ4_XS`, 17 GB) running locally on an NVIDIA GPU, working in a checkout of the AgenticLinux repository itself. Reasoning is switched off for speed.

<figure>
  <video controls muted playsinline preload="metadata" poster="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-claude.webp">
    <source src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-claude.webm" type="video/webm">
    <source src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-claude.mp4" type="video/mp4">
  </video>
  <figcaption><code>llmman launch claude --model qwen3.8</code>. Claude Code adds a shellcheck GitHub Actions workflow to the repo and verifies the four scripts pass, through llmman's Anthropic-compatible endpoint.</figcaption>
</figure>

<figure>
  <video controls muted playsinline preload="metadata" poster="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-opencode.webp">
    <source src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-opencode.webm" type="video/webm">
    <source src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-opencode.mp4" type="video/mp4">
  </video>
  <figcaption><code>llmman launch opencode --model qwen3.8</code>. OpenCode writes <code>hub-tags.sh</code>, which lists the image's Docker Hub tags with the architectures each one provides, and runs it.</figcaption>
</figure>

<figure>
  <video controls muted playsinline preload="metadata" poster="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-codex.webp">
    <source src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-codex.webm" type="video/webm">
    <source src="https://github.com/ericcurtin/ericcurtin.github.io/releases/download/assets/llmman-codex.mp4" type="video/mp4">
  </video>
  <figcaption><code>llmman launch codex --model qwen3.8</code>. Codex, over its Responses API, changes the OS itself: a weekly <code>docker system prune</code> timer added to the image's systemd units and enabled in <code>build.sh</code>. The change is a commit to the image, not a mutation of the running host.</figcaption>
</figure>

Three agents, three different wire protocols (Anthropic Messages, OpenAI Chat Completions, OpenAI Responses), one local model, no per-agent configuration. Swap `qwen3.8` for `qwen3.5:0.8b` and the same demo runs on a laptop CPU; the image's own smoke tests do exactly that on every CI run, on x86_64 and aarch64.

## Docker Sandboxes

[Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) run an agent in a microVM with its own kernel and a policy-controlled view of the host, which is the right shape for "let the agent loose on this repository". The `sbx` CLI is in the image, and it needs `/dev/kvm`, which any bare-metal install has.

Full support is due in the **next release**. On composefs-based systems with `/var/home` on its own mount, the current `sbx` (v0.40 through v0.43) creates the virtio-fs share for the workspace but the guest never mounts it, so the agent sees an empty directory and its writes are lost. That is tracked upstream as [docker/sbx-releases#597](https://github.com/docker/sbx-releases/issues/597); it affects every ostree/composefs Fedora variant, AgenticLinux included, and we will ship the fixed `sbx` as soon as it lands. Until then Docker Engine containers are the sandbox.

## Getting started

**Already on bootc or an ostree desktop** (Fedora Silverblue/Kinoite, Bluefin, Bazzite, ...):

```
sudo bootc switch docker.io/ericcurtin044/agenticlinux:kde   # or gnome, sway, cosmic, xfce, budgie, base
sudo reboot
```

**Fresh install:** download the network-installer ISO for your desktop and architecture from the [releases page](https://github.com/ericcurtin/agenticlinux/releases), write it to a USB stick and answer the installer's questions. The ISO fetches the image from Docker Hub during installation.

**Try it in a VM:** any KVM/HVF/WHPX hypervisor works; the CI boots each image in QEMU on Linux, macOS and Windows on every build.

After first login:

```
sudo usermod -aG docker "$USER"   # then log out and in again
llmman launch claude --model qwen3.5:0.8b   # a small model that runs anywhere
```

To change what is in the OS, change the [`Dockerfile`](https://github.com/ericcurtin/agenticlinux/blob/main/Dockerfile) and [`packages.txt`](https://github.com/ericcurtin/agenticlinux/blob/main/packages.txt), or `FROM docker.io/ericcurtin044/agenticlinux:kde` in your own image and `bootc switch` to it. Issues and pull requests are welcome at [github.com/ericcurtin/agenticlinux](https://github.com/ericcurtin/agenticlinux).
