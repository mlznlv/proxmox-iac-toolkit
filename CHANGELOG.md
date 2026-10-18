# Changelog

What changed for someone consuming this toolkit, in the format [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) describes, with versions following [Semantic Versioning](https://semver.org/).

This file records consumer-facing change. It does not restate platform targets or evidence, which belong to [Compatibility](docs/compatibility.md), or the rules governing releases, which belong to the [release policy](docs/release-policy.md).

## Unreleased

### Changed

- **Breaking: the `proxmox-linux-vm` module requires a DNS search domain.** The new `dns_search_domain` input has no default and takes one DNS name, such as `example.invalid`, which reaches the VM's cloud-init configuration unchanged. Until now the module declared none, so a clone kept its template's search domain, and a VM without one was given its node's. A configuration without the input now stops with `Missing required argument`, and a null, empty, whitespace, list, malformed or trailing-dot value, or a `name` and domain longer than 253 characters together, stops at plan with the module's message. Removing the domain later, or going back to the template's or node's, is not supported. To migrate:
  - Choose the domain before you move your pin, and add it to every call of the module. Nothing discovers the domain a VM has now, carries it over, or rewrites your configuration. To keep the one a VM uses today, look it up in its template's or node's configuration and declare it.
  - Plan and read the plan before you apply it explicitly. An existing VM should show `update in-place`, with only the DNS `domain` in `initialization` changing; a template's own domain, declared again, shows no change. Do not apply a plan that replaces the VM or changes anything else.
  - Expect disruption when you apply. The cloud-init drive is rebuilt and a running VM reboots into a new cloud-init instance, which can regenerate its SSH host keys and drop connections. The guest's fully qualified name can change even if you declare the node's own domain, because a VM that fell back to it was named without one. No path without that disruption is offered.

  The [module README](tofu/modules/proxmox-linux-vm/README.md#dns-search-domain) sets this out, and [Compatibility](docs/compatibility.md#vm-dns-search-domain) records the evidence behind it.
- **The repository is now `proxmox-iac-toolkit`,** renamed from `homelab-iac-toolkit`. Commits and tags are unchanged, so the full SHA you record still names the same source, and no module, role, input, output or behaviour changed. The worked example's checkout convention is now `vendor/proxmox-iac-toolkit/`. Update any Git remote or module `source` that names the old repository, keeping the same commit, and either keep your checkout path or change every reference to it together. No state change is involved. [Using the toolkit](docs/using.md#moving-from-homelab-iac-toolkit) has the commands.
- **The `proxmox-linux-vm` module rejects more malformed inputs when you plan.** `ipv4_address_cidr`, `ipv4_gateway` and IPv4 `dns_servers` must be dotted decimal without leading zeros, `ipv4_address_cidr` needs a prefix length from 8 to 32, and an IPv4-mapped IPv6 address is no longer read as IPv4. Each `ssh_public_keys` entry must be one key on one line, with at most two base64 padding characters. A configuration that used one of these values planned before and now stops at plan with the module's message; correct the value as the message says. `vm_id` and `network_vlan_id` left at `null` no longer fail validation on older OpenTofu releases. [Compatibility](docs/compatibility.md#opentofu-cli-compatibility-and-evidence) records the versions checked.

### Fixed

- **The `proxmox-linux-vm` module sends each `ssh_public_keys` entry without its surrounding whitespace.** It used to send each entry as given. The locked provider trims only the ends of the joined keys when it reads them back, so whitespace before the first key or after the last, such as the newline a key read with `file()` ends in, or a newline ending any other key, planned an in-place update on every run, and by the provider's source each such apply rebuilds the cloud-init drive and reboots a running VM. The keys the module accepts are unchanged and your configuration needs no change, but an existing VM may need one apply. To migrate:
  - Expect no change for a VM whose keys, as the provider read them back, already match your keys trimmed: one key, or several with whitespace only before the first or after the last.
  - Expect one in-place update that changes only the bootstrap `keys` whenever a key other than the last ends in whitespace or a key other than the first starts with it. Once it is applied, the next plan proposes nothing. This includes configurations that planned no change before: spaces or tabs at the start of a key other than the first, or at the end of a key other than the last, were read back as sent. A newline ending a key other than the last, as with `[file("a.pub"), file("b.pub")]`, already planned that update on every run, and plans it one last time.
  - Plan and read the plan before you apply it explicitly. The VM should show `update in-place`, with only `keys` changing. Do not apply a plan that replaces the VM or changes anything you do not expect.
  - Expect disruption when you apply. The cloud-init drive is rebuilt and a running VM reboots, which drops connections. If the user data Proxmox generates changes, as removing spaces, tabs or a carriage return from a key can make it, the guest can boot a new cloud-init instance, re-apply its bootstrap configuration and regenerate its SSH host keys. Removing only an empty key can leave that data unchanged, but the reboot still happens. No path without it is offered, and nothing here promises the guest stays reachable.

  The [module README](tofu/modules/proxmox-linux-vm/README.md#bootstrap-ssh-keys) sets this out, and [Compatibility](docs/compatibility.md#vm-bootstrap-ssh-keys) records the evidence behind it: plans against a model of what the provider reads back, not observations on a Proxmox VE ([#133](https://github.com/mlznlv/proxmox-iac-toolkit/issues/133)).

## 0.1.0-alpha.1 - 2026-09-08

### Added

- **An OpenTofu module, `proxmox-linux-vm`,** that creates one Proxmox VE Linux VM as a full clone of a cloud-init template you supply. It takes your node, template, datastore, sizing, bridge, static IPv4 addressing, and a bootstrap account, and it configures no provider and declares no backend, so both stay yours. It publishes a non-sensitive `connection` output — host, user and port — for you to compose inventory from by hand.
- **An Ansible role, `qemu_guest_agent`,** that installs the QEMU guest agent in a Linux guest and brings its service up. It takes ordinary inventory, reads no OpenTofu state, and manages no Proxmox resource, so it works against any host you can already reach.
- **A coordinated way to consume both together:** one checkout of this repository at one immutable full commit SHA, recorded in your own source control, with the module resolved through a local path into it and Ansible's `roles_path` reaching the same checkout. One commit therefore determines both component implementations.
- **A worked example**, `examples/separate-consumer-repository/`, showing that arrangement as a separate repository would hold it, including the manual mapping from the module's `connection` output into ordinary inventory.
- **Consumer documentation**, [Using the toolkit](docs/using.md), covering the workflow, what you own, and what to check when a checkout, a role path, or an input does not resolve.

### Notes for consumers

- **The guest-agent channel is attached when the VM is created**, which is what lets the role start the service on its first run. Turning it off produces a VM where the agent cannot run, and reversing that costs a stop and start.
- **Destroy stops the VM by default** rather than asking the guest to shut down. That is predictable, and it can interrupt a running workload and lose unwritten data. The [module README](tofu/modules/proxmox-linux-vm/README.md) sets out both settings.
- **No validation in this repository has run against a Proxmox VE or a guest.** Every check is credential-free and structural. [Compatibility](docs/compatibility.md) records what that evidence does and does not demonstrate.

### Migration

None. `0.1.0-alpha.1` is the initial release of this toolkit, so there is no earlier release to migrate from.
