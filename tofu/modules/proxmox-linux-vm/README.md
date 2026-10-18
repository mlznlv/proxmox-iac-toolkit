# `proxmox-linux-vm`

Create one full-clone Linux VM from a consumer-supplied cloud-init template. The module owns its
Proxmox lifecycle and initial access bootstrap. Provider configuration, backend/state, templates
and continuing guest configuration are consumer-owned. See [Using the
toolkit](../../../docs/using.md).

## Requirements

[Compatibility](../../../docs/compatibility.md#support-and-evidence-matrix) records the Proxmox
VE 9.x target and [sampled CLI
evidence](../../../docs/compatibility.md#opentofu-cli-compatibility-and-evidence). Executable
requirements are in [`versions.tf`](versions.tf); your root lock selects the provider build.

Supply a template on the target node whose cloud-init handles your selected addressing, DNS,
username and SSH public keys. Additional addressed interfaces must be matched by MAC; DHCP needs
a functioning DHCPv4 renderer/client; static IPv6 needs its address/gateway renderer. The
template need not contain the agent.

The clone inherits template disks and configuration. The template or Proxmox's default chooses
cloud-init format and per-instance package upgrade behavior; the template's default-user
configuration chooses bootstrap groups/sudo rules. The VM's DNS search domain is the one you
declare, not the template's or node's; see [DNS search domain](#dns-search-domain).

You supply bridges, VLAN transport/switching/routing, suitable addresses and DNS; the DNS search
domain you declare, the namespace and DNS service behind it, and a domain that fits with the
VM's name; DHCP service/reservations/naming; IPv6 prefix allocation, uniqueness, routing,
upstream service and any router advertisements; and SSH private keys. The module cannot check
that an address/gateway belongs on a bridge or VLAN.

## Quick start

In a consumer root using your pinned checkout:

```hcl
module "fictional_vm" {
  source = "./vendor/proxmox-iac-toolkit/tofu/modules/proxmox-linux-vm"

  name           = "fictional-vm"
  node_name      = "fictional-node"
  template_id    = 9000
  datastore_id   = "fictional-datastore"
  cpu_cores      = 2
  memory_mib     = 2048
  network_bridge = "vmbr0"

  ipv4_address_cidr = "192.0.2.10/24"
  ipv4_gateway      = "192.0.2.1"
  dns_servers       = ["192.0.2.53"]
  dns_search_domain = "example.invalid"
  username          = "fictional"
  ssh_public_keys   = [trimspace(file(pathexpand("~/.ssh/id_ed25519.pub")))]
}
```

Values are fictional or reserved for documentation;
[Security](../../../SECURITY.md#publication-boundary) names the reserved ranges. Supply
provider/backend configuration, then follow the [reviewed-plan
workflow](../../../docs/using.md#workflow).

## Inputs

Required:

| Name | Type | Description |
| --- | --- | --- |
| `name` | `string` | VM name. Proxmox requires a valid DNS label. |
| `node_name` | `string` | Proxmox node to create the VM on. The template must be on this node. |
| `template_id` | `number` | Identifier of the cloud-init template to clone. |
| `datastore_id` | `string` | Datastore for the cloned disks and the cloud-init drive. |
| `cpu_cores` | `number` | CPU cores. |
| `memory_mib` | `number` | Dedicated memory, in MiB. |
| `network_bridge` | `string` | Bridge for the VM's primary network device, `net0`, such as `vmbr0`. |
| `ipv4_address_cidr` | `string` | Static IPv4 address in dotted decimal with a prefix length from 8 to 32, such as `192.0.2.10/24`. Required in static mode, the default; must be null with [DHCP](#dhcp-addressing). |
| `ipv4_gateway` | `string` | IPv4 default gateway in dotted decimal, without a prefix length. Required in static mode; must be null with DHCP. |
| `dns_servers` | `list(string)` | IPv4 or IPv6 DNS servers, in order of preference. At least one. |
| `dns_search_domain` | `string` | DNS search domain for the guest: one DNS name, such as `example.invalid`, without a trailing dot. There is no default and no value for no search domain. See [DNS search domain](#dns-search-domain). |
| `username` | `string` | Bootstrap account cloud-init creates. |
| `ssh_public_keys` | `list(string)` | OpenSSH **public** keys authorized for that account. At least one, each one key on one line; a trailing newline is allowed. Each key is sent without its surrounding whitespace. See [Bootstrap SSH keys](#bootstrap-ssh-keys). |

Optional:

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `vm_id` | `number` | `null` | VM identifier. Proxmox assigns the next free one when this is null. |
| `guest_agent_enabled` | `bool` | `true` | Whether to attach the guest-agent channel. Off produces a VM where the agent cannot run. |
| `stop_on_destroy` | `bool` | `true` | Whether destroy stops the VM instead of asking the guest to shut down. |
| `ssh_port` | `number` | `22` | Port published in the `connection` output. Metadata only. |
| `network_vlan_id` | `number` | `null` | Access VLAN tag for the primary network device, `1` to `4094`. Null leaves it untagged. |
| `ipv4_mode` | `string` | `"static"` | IPv4 addressing mode of the primary attachment: `"static"` or `"dhcp"`. |
| `connection_host` | `string` | `null` | Host published in the `connection` output: a hostname or an IPv4 or IPv6 address. Required with DHCP. Metadata only. |
| `network_mac_address` | `string` | `null` | MAC address for the primary network device, `net0`. Null lets Proxmox assign one. |
| `additional_network_attachments` | `list(object)` | `[]` | Up to seven more network attachments, `net1` onwards, in the order given. On an existing VM they can be added and changed, not removed or reordered. |
| `ipv6_address_cidr` | `string` | `null` | Static IPv6 address with a prefix length from `8` to `128`, such as `2001:db8::10/64`, added to the primary attachment beside its IPv4 addressing. Null declares no IPv6. |
| `ipv6_gateway` | `string` | `null` | IPv6 default gateway, global or link-local, without a prefix length. Needs `ipv6_address_cidr`. |

Locally decidable invalid inputs fail before apply. Numeric ranges, DNS-label names, key syntax,
address families/prefixes, modes, conditional requirements, and duplicate declared IP/MAC
addresses are checked in [`variables.tf`](variables.tf) and [`main.tf`](main.tf). Runtime
prerequisites remain the consumer's responsibility. IPv4 addresses, gateways and DNS servers
reject leading-zero octets; IPv4-mapped IPv6 is not accepted as IPv4. SSH-key base64 allows at
most two padding characters. A DNS search domain that is null, empty or contains whitespace,
ends with a dot, is longer than 253 characters, has a label longer than 63 characters or one
that is not letters, digits and inner hyphens, or names more than one domain is rejected, as is
a name and search domain longer than 253 characters together.

## Outputs

| Output | Value |
| --- | --- |
| `connection` | `{ host, user, port }`: supplied `connection_host`, otherwise static IPv4 without prefix; bootstrap username; `ssh_port` |
| `mac_addresses` | All declared/provider-assigned MACs in slot order, `net0` first |
| `vm_id` | Supplied or assigned VM identifier |
| `vm_name` | Declared name |

All are non-sensitive; the module never publishes a private key or a password, and never accepts
one. `connection` is metadata: it configures no SSH service, resolves no hostname, discovers no
address and proves no reachability. Declared IPv6 never becomes `host` automatically. Map values
into ordinary inventory under [Using](../../../docs/using.md#workflow); no component reads them
at runtime.

## Network examples

These are input fragments for the complete call above. An input the call already has, such as
`dns_servers`, is replaced rather than repeated.

### VLAN tagging

```hcl
network_vlan_id = 42
```

`null` means untagged while keeping the device attached. Only whole tags 1–4094 are accepted;
provider sentinel `0` is not public. No VLAN/bridge creation, trunking or guest VLAN interface
is configured.

### DHCP addressing

Replace the static IPv4 inputs with:

```hcl
ipv4_mode           = "dhcp"
connection_host     = "fictional-vm.example.invalid"
network_mac_address = "00:00:5E:00:53:01"
```

Static mode requires address and gateway; DHCP requires both null and an explicit connection
host. DNS servers remain required in both. A connection host may be a hostname or plain
IPv4/IPv6 literal, without scheme, brackets or port. Ambiguous IPv4 spellings such as
leading-zero octets or hexadecimal are rejected.

For a reservation effective at first boot, declare a MAC and create the reservation before
apply. Otherwise Proxmox assigns one and publishes it later. Apply waits for neither a lease nor
an agent. Additional attachments remain static or addressless, avoiding multiple DHCP-supplied
default routes.

### Static IPv6

```hcl
ipv6_address_cidr = "2001:db8::10/64"
ipv6_gateway      = "fe80::1"
dns_servers       = ["2001:db8::53", "192.0.2.53"]
```

IPv6 is optional beside either primary IPv4 mode. The address requires a prefix 8–128; a global
or link-local gateway is optional but requires the address. `auto`, `dhcp` and `manual` are
rejected when you plan; if your network relies on autoconfiguration or DHCPv6, configure IPv6
inside the guest instead. IPv6-only and additional-attachment IPv6 are unsupported. Without a
gateway the module makes no default-route claim. Removing declared IPv6 does not remove
guest-configured link-local or router-advertisement addresses.

### Additional network attachments

**An attachment you add is one you keep.** Read [Attachment retirement is
unsupported](#attachment-retirement-is-unsupported) before you declare any.

```hcl
additional_network_attachments = [
  { bridge = "vmbr1", vlan_id = 42, ipv4_address_cidr = "198.51.100.10/24" },
  { bridge = "vmbr2", mac_address = "00:00:5E:00:53:02" },
]
```

Write `[]`, or omit the input, for none. A `null` is refused rather than read as an empty list, so
one passed down by a wrapping module cannot ask to remove attachments a VM already has.

| Attribute | Contract |
| --- | --- |
| `bridge` | Required bridge |
| `vlan_id` | Optional access tag; same rules as primary |
| `ipv4_address_cidr` | Optional dotted-decimal static IPv4, prefix 8–32; null declares no IP configuration |
| `mac_address` | Optional unicast MAC, six colon-separated hex octets; passed unchanged |

Position is identity: entries map to `net1` onwards with matching cloud-init slots, at most
seven additions/eight attachments total. Proxmox network data matches addressed devices by MAC
and names them `eth<slot>`. An undeclared MAC can appear as null in the first plan; read the
assigned value after apply. Declared MAC uniqueness is case-insensitive across all attachments;
assigned, retained and network-wide collisions cannot be checked locally.

Only the primary has a gateway. Extra object keys are silently discarded by OpenTofu: `gateway`,
IPv6 fields or a typo such as `vlan` do nothing. Review the actual planned fields. Addressless
means no configuration declared by this module: inherited template `ipconfig` entries, earlier
slot addresses or guest configuration may still supply one.

## DNS search domain

`dns_search_domain` is the search domain the guest is given, declared in the VM's cloud-init
configuration exactly as you write it. It is required beside `dns_servers`, and it has no
default: the domain, the namespace it names, and the DNS service behind it are yours, and the
module will not choose one for you. The [Quick start](#quick-start) declares `example.invalid`,
which is reserved by [RFC 2606](https://www.rfc-editor.org/info/rfc2606/) and names nothing
real.

It takes one DNS name: labels of ASCII letters, digits, and inner hyphens, each at most 63
characters, separated by dots, at most 253 characters in all, and with no trailing dot. A single
label, such as `test`, is accepted. A null, an empty value, whitespace anywhere, a list of
domains in any form, and `.` alone are rejected when you plan, each with a message naming the
rule, and there is no value that asks for no search domain.

**A template's or node's search domain no longer applies.** Without a declaration, a clone keeps
whatever search domain its template has, and Proxmox gives a VM that has none the search domain
of the node it runs on, even though the DNS servers are yours. Declaring one makes the domain
yours in both cases. [ADR 0019](../../../docs/decisions/0019-explicit-vm-dns-search-domain.md)
records why.

**Proxmox also uses it to name the guest.** It joins the VM's `name` and the search domain into
the guest's fully qualified name, `name.domain`. The two together must fit in 253 characters,
and a combination that does not is rejected when you plan, even when each is valid alone. `name`
stays one DNS label: the search domain does not let you write a qualified name there. The
`connection` output never has the domain appended; publish a name through `connection_host` if
you want one.

### Declaring or changing the domain on an existing VM

**This input is a breaking addition.** A configuration written for a revision without it stops
at `Missing required argument` until it declares one. Before you move your pin to a revision
that has it, choose a domain and add it to every call of this module. Nothing discovers the
domain a VM has now, carries it over, or rewrites your configuration. If you want to keep the
domain a VM uses today, look it up in its template's or node's configuration and declare it;
that is a choice you make, not a default.

Declaring a search domain on a VM created without one, or changing it to another, updates the VM
in place; it does not replace it. With the provider build this repository locks, the plan
changes only the search domain in the VM's cloud-init configuration, and once that update is
applied the next plan proposes nothing. That holds whether the VM had no search domain of its
own or one its template gave it. Declaring the domain the template already gave it plans no
change at all.

It is still disruptive, as any change to the cloud-init configuration is. It rebuilds the
cloud-init drive and reboots a running VM, and the changed data makes the next boot a new
cloud-init instance. Cloud-init re-applies its configuration and, unless your template's
configuration says otherwise, deletes and regenerates the guest's SSH host keys, so expect a
host-key warning and connections to drop. This module offers no way to make the change without
that disruption. The guest's fully qualified name can change too, even when the domain does not
seem to: a VM that took the node's search domain was named without one, so declaring that same
domain gives it `name.domain` for the first time.

Read the plan before you apply it, and apply it explicitly. Declaring or changing the domain
shows the VM as `update in-place`, with only the DNS `domain` in `initialization` changing. If a
plan shows the VM being replaced, or anything else changing, do not apply it: that is not the
behaviour this module relies on. Check which provider build your root module's lock selects,
because only the build named below has been evaluated.

### Removing the domain is not supported

Removing the search domain, setting it empty, returning a VM to its template's or node's search
domain, and running it without one are not supported, and the module refuses a null or empty
value rather than asking for any of them. The locked provider leaves an empty domain out of its
request instead of clearing the one Proxmox holds, and reads that one back, so such a change
would never settle: every plan would propose it again. Choose a domain you mean to keep.
Declaring the value a template or node supplied is an ordinary change of domain, and the VM then
keeps it as declared.

### Where the search-domain behaviour is evidenced

All of the above is for the provider build this repository locks, `bpg/proxmox` v0.111.1, at
provider level. Credential-free plans made with that build, with refresh disabled, against
synthetic prior states covered creating a VM with a declared domain, declaring one on a VM that
had none and on one whose template gave it another, and changing one declared domain to another,
each in static and DHCP modes and beside static IPv6 and an additional attachment. Each was then
planned again against a state modelled on the provider's read path, and proposed nothing; no
apply and no refresh was run. The inheritance, naming and empty-domain behaviour above is read
from the provider's and Proxmox's source.

None of it has been observed against a real Proxmox VE. Nothing here shows that a guest's
resolver uses the domain, that its hostname or fully qualified name changes, that the reboot
succeeds, that host keys are regenerated, or that the guest is reachable afterwards.
[Compatibility](../../../docs/compatibility.md#vm-dns-search-domain) records that boundary, and
your root module's own lock decides which provider build you run.

## Bootstrap SSH keys

`ssh_public_keys` are the keys cloud-init authorizes for the bootstrap account. Each entry is
one OpenSSH public key on one line, with or without a comment, and whitespace around it is
allowed, such as the newline a key read with `file()` ends in. The module sends each entry
without that whitespace, keeping the order you declare, any duplicates, and each key and comment
exactly as written.

The provider build this repository locks joins the keys with newlines when it sends them. When
it reads them back, it trims whitespace from the ends of the joined value only, then splits it
on newlines. Keys sent untrimmed could therefore read back differently from what was sent:
whitespace before the first key or after the last was lost, and a newline ending any other key
became an extra, empty key. Either planned an update on every run. Sent trimmed, the keys read
back as sent, and the next plan proposes nothing.

### Updating a VM created before keys were trimmed

**Your configuration needs no change, but an existing VM may need one apply.** After you move
your pin, the first plan for an existing VM proposes nothing when what it holds already matches
your keys trimmed. That is the case for one key, and for several keys with whitespace, if any,
only before the first or after the last.

Otherwise, whenever a key other than the last ends in whitespace or a key other than the first
starts with it, the first plan proposes one update in place that changes only the bootstrap
`keys`. It removes the empty key or the whitespace left on a key, and keeps the keys, their
order, their duplicates and their comments. Once that update is applied, the next plan proposes
nothing. Two kinds of configuration get it:

- **Ones that already planned an update on every run:** a newline at the end of a key other than
  the last, as with `[file("a.pub"), file("b.pub")]`, or at the start of a key other than the
  first. The update is proposed one last time.
- **Ones that planned no change before:** other whitespace, such as spaces or tabs, at the start
  of a key other than the first or at the end of a key other than the last. The provider read it
  back as it was sent, so these were stable, and they now plan this update once.

**The update is disruptive.** By the provider's source, any change to the cloud-init
configuration rebuilds the cloud-init drive and reboots a running VM, so schedule it, and expect
connections to drop. What the guest does next depends on whether the user data Proxmox generates
for it changes. Removing only an empty key can leave that data as it was. Removing spaces, tabs
or a carriage return from a key can change it, and changed data can make the next boot a new
cloud-init instance, which re-applies cloud-init's configuration, the bootstrap account
included, and, depending on your template's configuration, can regenerate the guest's SSH host
keys, so a host-key warning may follow. This module offers no way to make the update without the
reboot, and promises nothing about reaching the guest afterwards.

Read the plan before you apply it, and apply it explicitly. The update shows the VM as
`update in-place`, with only `keys` in the bootstrap account changing. If a plan shows the VM
being replaced, or anything else changing that this README or the [changelog](../../../CHANGELOG.md)
does not tell you to expect, do not apply it. Nothing in this module applies the update for you
or edits state, and the guest's authorized keys after cloud-init stay yours to manage.

### Where the key behaviour is evidenced

All of the above is for the provider build this repository locks, `bpg/proxmox` v0.111.1. The
plans were made with that build, with refresh disabled, against prior states written through a
model of how it sends and reads back the keys and of Proxmox storing them as sent, and each
update was planned again against its modelled read-back. The send and read paths, the reboot,
and how Proxmox generates user data are read from the provider's and Proxmox's source.

None of it has been observed against a real Proxmox VE. Nothing here shows the reboot, a new
cloud-init instance, host keys regenerated, or that the guest is reachable afterwards.
[Compatibility](../../../docs/compatibility.md#vm-bootstrap-ssh-keys) records that boundary, and
your root module's own lock decides which provider build you run.

## Lifecycle

These are provider-level expectations, not observed live behavior; the exact build and evidence
are in [Compatibility](../../../docs/compatibility.md#provider-lifecycle-evidence). Read every
plan before explicit apply. Unexpected replacement for a supported update is a stop condition
requiring provider/Architecture review.

| Change | Contract and consequence |
| --- | --- |
| Add/change/remove a VLAN; change an additional attachment's `bridge` | In-place; running link detaches/re-attaches and interrupts connectivity; whether it returns depends on your bridge, VLAN, switching, routing and addressing, not on this module |
| Append an additional attachment | In-place; new device; existing slots/primary MAC preserved; an addressed one also has the initialization consequences below |
| Change or first declare a MAC | In-place VM update; old device is unplugged and a different one appears; old reservations stop matching. The same address in a different letter case plans as a change too |
| Remove a MAC declaration | Retains the current address; no change |
| Add/change an attachment address, including the primary's own; switch static/DHCP | In-place, with only `ipconfig0` changing for a mode switch; initialization consequences below |
| Add/change/remove primary IPv6 or change DNS | In-place, with only `initialization` changing for an IPv6-only change; initialization consequences below |
| Declare or change the DNS search domain | In-place, with only the DNS `domain` in `initialization` changing; initialization consequences below. Removal is unsupported; see [DNS search domain](#dns-search-domain) |
| First plan for a VM created before keys were trimmed, where what it holds differs from the trimmed keys | One in-place update changing only `keys`, then no change; reboot and possible new cloud-init instance as in [Bootstrap SSH keys](#bootstrap-ssh-keys) |
| Change only connection host/port | Output only; no infrastructure change |
| Change source template | Replacement destroys the VM and its disks |

Initialization changes rebuild cloud-init and reboot a running VM on the pinned path. Changed
data creates a new cloud-init instance: network configuration is reapplied and, unless the
template's cloud-init disables it, SSH host keys are deleted and regenerated, so expect a host-key
warning on the next connection. Review package-upgrade/default-user behavior
too. Update dependent addresses, DNS records, inventory and known host keys. An addressless
appended device alone leaves cloud-init data unchanged. Supported updates must converge to a
subsequent no-change plan; an initial in-place plan alone is insufficient evidence.

### Attachment retirement is unsupported

Do not remove/reorder attachments, clear the list, remove an existing address, or remove then
re-add a device, including an addressless final device. Current input validation cannot see
history, so these can plan as ordinary in-place updates. **Reject plans removing devices/IP
configurations or remapping slots.**

The provider leaves retired `ipconfigN` entries and reads them back, causing recurring
differences and repeated cloud-init rebuild/reboot requests. A later device can inherit a
retired address. Assigned MACs stay with slots; declared MACs move with reordered entries. A
removal relinquishes its MAC and re-addition promises neither reuse nor a different value. Even
cases that happen to settle remain excluded uniformly.

Repeated apply, ignored differences, disabled reboot or direct API cleanup are not remedies.
Keep slots, wait for an evidenced provider remedy, or plan a deliberate reviewed replacement
with data preservation. Replacement destroys the old VM and disks. [ADR
0014](../../../docs/decisions/0014-additional-attachment-retirement-lifecycle.md) explains the
restriction and the evidence needed to change it.

### Guest agent and destroy

The channel is attached by default and provider IP waiting is always disabled. Create the VM,
compose inventory, then run the [guest-agent
role](../../../ansible/roles/qemu_guest_agent/README.md); no second apply or power cycle is
needed. Opting out creates a VM where the agent cannot run; attaching the channel later requires
stop/start, because it is not hot-plugged. [ADR
0006](../../../docs/decisions/0006-guest-agent-channel-at-creation.md) records it in full.

Before the agent runs, shutdown/reboot probe it for three seconds, warn and fall back to ACPI;
queries report unavailable. Backups skip filesystem freeze and are crash-consistent rather than
filesystem-consistent. Close this window before relying on agent-dependent operations or
filesystem-consistent backups.

`stop_on_destroy = true` stops the VM without ACPI/agent dependence and can lose unwritten data
or interrupt workloads. `false` requests shutdown via ACPI before the agent runs, or via the
agent afterwards; it can block or time out. Choose that override only when accepting this
trade-off. Destroy remains destructive.

## Limitations and validation

No template acquisition, linked/cross-node clones, disk mutation, DHCP service or address
discovery, extra gateways/routes, configurable NIC models, firewall, MTU/rate/queue/link state,
guest network management, HA or pools are included. Nor does it support more than one DNS search
domain, no search domain, a search domain left to the template or node, or a qualified VM name.
Continuing packages, users, keys and SSH configuration belong to the consumer. For containers
use the [separate module](../proxmox-linux-container/README.md).

[`tests/`](tests) checks declarations and invalid inputs with a mocked provider. Those checks
run on the contributor OpenTofu pin in `.tool-versions`, not at this module's consumer minimum,
and the repository lock fixes their provider build.
[Contributing](../../../CONTRIBUTING.md#validate-your-change) owns commands and required
evidence; [Compatibility](../../../docs/compatibility.md#current-evidence-level) owns what has
been established. The module lock does not constrain your root lock.
