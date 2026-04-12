# task-nixos-mgmt

A simple [Taskfile](https://taskfile.dev)-based tool for managing small fleets of NixOS hosts. Build, test, deploy, and operate NixOS configurations across multiple machines. Uses [`go-task`](https://taskfile.dev), SSH and `nixos-rebuild` under the hood.

Operations are done in serial manner so the fleet size is limited by your patience. 

![Demo](pics/demo.gif)

> ⚠️ **Not a replacement for large-scale fleet management tools**
> This is designed for **small to medium-sized NixOS fleets** (e.g. homelabs, small infra, edge nodes).
> It is **not intended to replace tools like [Colmena](https://colmena.cli.rs/)** or other large-scale fleet orchestration systems.

## ✨ Features

* Multi-host NixOS management via simple `task` commands
* Local + remote host handling with unified interface
* Built-in safety mechanisms for remote deployments (test, rollback or switch)
* Git-based workflow support
* SSH-based execution (`ssh-agent` strongly recommended)
* Minimal dependencies and easy to understand logic


## ⚙️ Requirements

### 1) Install Taskfile

Add `go-task` to your `environment.systemPackages` to install [Taskfile](https://taskfile.dev).

```nix
  environment.systemPackages = with pkgs; [
    go-task
  ];
```

### 2) Set up key-based SSH access to all hosts

All managed hosts must be accessible via SSH.

* SSH key-based authentication is **required**
* An SSH agent (e.g. `ssh-agent`) is **strongly recommended**
* SSH agent forwarding support (`ssh -A`), please ensure it is enabled in your environment.

Typical setup:

```bash
ssh-keygen -t ed25519
ssh-copy-id user@host
eval "$(ssh-agent)"
ssh-add ~/.ssh/id_ed25519
```

### 3) Configure hosts into `hosts.yml`

You must define the hosts you want to manage in `hosts.yml`.

Example:

```yaml
version: '3'

vars:
  HOSTS:
    - host1
    - host2
    - host3
    - another_host.domain
```

Optional: define SSH aliases to use short host names:

```yaml
vars:
  HOST_ALIASES:
    host1: host1.domain.example
    host2: host2.domain2.example
    host3: 10.0.0.100
```

## 4) Set up repository structure


```text
# /etc/nixos
.
├── Taskfile.yml
├── hosts.yml
├── hosts/
│   └── <hostname>/
│       └── configuration.nix
```

Each host:

* Must exist in `hosts.yml` in `HOSTS` list
* Must have a corresponding directory under `hosts/`
* Must contain a valid `configuration.nix`

Example:

```text
hosts/
├── host1/
│   └── configuration.nix
├── host2/
│   └── configuration.nix
```

## 🚀 Getting Started

### Validate configuration

```bash
task eval
```

Checks syntax using `nix-instantiate` 

### Dry build (safe pre-check)

```bash
task check
```

Runs `nixos-rebuild dry-build` for all hosts 

### Deploy configuration

```bash
task switch
```

Aliases: `task deploy`, `task build`

Applies configuration locally or remotely depending on host 

### Test configuration (temporary)

```bash
task test
```

Runs `nixos-rebuild test` (non-persistent) 

### Update channels

```bash
task update
```

### Run an arbitrary command

```bash
task cmd CMD="df -h" [-- HOST1 HOST2 ...]
```

### Check system status

```bash
task status
```

Checks `systemd status` on the hosts and compares NixOS configuration timestamps to the `/hosts/<host>/.rebuild_time`. 

### Git workflow

```bash
task push        # push repo
task pull        # pull on hosts
task push-pull   # both
```

## 🎯 Targeting Specific Hosts

```bash
task switch -- host1 host2
```

If no hosts are specified, all hosts are targeted.


## 🔐 Safe Remote Deployment Logic

One of the key design goals of this project is **safe remote management**.

The Taskfile implements a **multi-step deployment safety mechanism**:

### 1. SSH connectivity check

* Verifies host is reachable before deployment

### 2. Automatic rollback timer

A rollback reboot is scheduled in 10 minutes:

```bash
shutdown -r +10
```

If something goes wrong, the machine will reboot into the previous generation.

### 3. Test deployment (`nixos-rebuild test`)

* Applies configuration temporarily
* Does **not** make it default

### 4. SSH re-check

* Ensures system is still reachable via SSH after applying config

### 5. Cancel rollback

If everything is OK:

```bash
shutdown -c
```

### 6. System health validation

```bash
systemctl is-system-running
```

* If **degraded → rollback is triggered**
* If **healthy → continue**

### 7. Final switch

```bash
nixos-rebuild switch
```

Only executed after all checks pass.


## 🧠 Why This Matters

This approach prevents:

* Locking yourself out via SSH
* Deploying broken configurations
* Leaving systems in degraded states
* Risky blind `switch` operations

## 🆚 When to Use Something Else

Use tools like Colmena when:

* Managing **several tens or hundreds of hosts**
* Needing **parallel orchestration**
* Requiring **stateful deployments or secrets distribution at scale**

## 🧩 Design Philosophy

* Keep it simple and transparent
* Prefer shell + Taskfile over complex frameworks
* Make failures visible and actionable

## 📝 Notes

* Uses SSH agent forwarding (`ssh -A`)
* Assumes Git-based configuration management
* Works without flakes (compatible with traditional Nix setups)
* Includes a small spinner utility for better CLI UX. This is generated automatically to task

## 📄 License

[The MIT License (MIT)](https://mit-license.org/)

