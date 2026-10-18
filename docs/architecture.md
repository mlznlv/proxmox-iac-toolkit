# Architecture

## Repository boundary

The toolkit owns reusable source, interfaces, sanitized examples, documentation, validation and
contributor tooling. It must work without an unpublished file or private deployment repository.
Consumers own environment composition, inventory, provider/backend configuration, state custody,
credentials, identities, topology, sizing and workload policy. [Security](../SECURITY.md) owns
publication rules.

## Lifecycle ownership

| Concern | Owner |
| --- | --- |
| Proxmox resources and infrastructure dependencies | OpenTofu |
| Continuing guest OS, users, packages and services | Ansible |
| Transparent developer workflow orchestration | Task |
| Encryption interface for committed secrets | SOPS and age |
| Durable source, configuration, documentation and decisions | Git |
| Public validation | Repository tooling and GitHub Actions |

Each concern has one lifecycle owner. OpenTofu may bootstrap initial access; Ansible must not
create or destroy Proxmox resources. Scripts and CI invoke owners without maintaining competing
state, hidden recovery or configuration. See [ADR
0002](decisions/0002-lifecycle-orchestration-ownership.md).

## Dependency direction

Consumer configuration depends on reusable components. Components do not depend on examples,
private files, each other, or tool-specific assistant configuration. The VM and LXC modules are
siblings with separate interfaces; the guest-agent role takes ordinary inventory and reads no
OpenTofu state. Current interfaces, lifecycle rules, destructive or replacement-sensitive
behavior and limits belong to their READMEs, linked from the root index; every component
documents them.

[Using the toolkit](using.md) owns the coordinated checkout and manual handoff workflow. Task
does not acquire toolkit revisions, generate inventory, wire outputs, load credentials, or
automate deployment. Public examples must parse with fictional values and explicit input
boundaries; they declare no backend or provider endpoint and add no acquisition, state-reading
or secret mechanism. Example checks may use a credential-free fixture for the shared checkout but
must not introduce a second canonical consumption model for CI convenience; acquisition tooling
enters normal CI only if an accepted example owns it.

## Repository layout

| Area | Responsibility |
| --- | --- |
| `tofu/modules/<name>/` | Module source, interface README and contract tests |
| `ansible/roles/<name>/` | Role source, interface README and contract tests |
| `examples/<name>/` | Supported cross-component examples and their checks |
| `docs/decisions/` | Durable decision rationale |
| `.devcontainer/` | Canonical contributor environment |
| `.github/` | CI, dependency automation and contribution templates |
| `scripts/` | Small portable helpers when direct commands are insufficient |
| Root declarations/configuration | Shared tool versions, locks and validation |
| `.claude/` | Optional, non-authoritative agent assistance |

Component examples stay beside their component; cross-component examples live under `examples/`.
A component test fixture may demonstrate connection-output composition, but it must not become a
consumer example, read real state, or couple the Ansible role to OpenTofu.
Add hierarchies only for approved content. No speculative platform, distribution, service,
environment or shared abstraction is implied.
[Contributing](../CONTRIBUTING.md#documentation-ownership) owns document placement.

## Contributor control plane

The Dev Container, native setup and CI consume the same executable declarations.
[Contributing](../CONTRIBUTING.md) owns setup and evidence procedures;
[Taskfile.yml](../Taskfile.yml) owns discoverable, direct commands. Task uses no cached
validation result. Optional assistant configuration must be removable without losing correctness
or usability and must link to public authority.

## Validation and live-testing boundary

Normal local and public CI validation is deterministic, non-destructive and credential-free,
with the same checks, versions, configuration and pass/fail semantics. Public network access
serves dependency acquisition and link checks; private networks, repositories and deployment
infrastructure are excluded.

CI treats PR code as untrusted, uses minimal token permissions, persists no credentials unless
an approved workflow needs them, pins third-party actions immutably and verifies downloaded
executables/dependencies. Privileged PR triggers must not execute untrusted code. No sensitive
state, plans, logs or decrypted artifacts may be published. [ADR
0004](decisions/0004-local-validation-task-ci-security-boundary.md) records these decisions;
[Compatibility](compatibility.md) owns evidence claims.

Live testing requires separate accepted Architecture, isolation from normal CI, explicit opt-in
and defined credential custody, trusted triggers, infrastructure ownership, cleanup, failures
and cost responsibility before implementation. A new ADR is required when that design changes a
durable security, lifecycle-ownership, compatibility or validation boundary; implementation
details within an accepted design, such as its runner, schedule or trigger mechanics, may change
without a new ADR when those boundaries remain unchanged. Proposed [ADR
0018](decisions/0018-isolated-live-validation-path.md), tracked in
[#105](https://github.com/mlznlv/proxmox-iac-toolkit/issues/105), does not itself authorize a
live run.

## Extension boundaries

[ADR 0017](decisions/0017-guest-distribution-extension-and-qualification.md) governs separate
VM-template, LXC and per-role qualification. Support is not transitive between tracks or to
their composition. An unchanged suitable VM template is a consumer choice; naming a target is a
claim; changing bootstrap is a capability decision. Debian LXC has no distribution extension
point. Another target needs its own decision, including placement; no OS selector or
distribution layer is introduced.
[Compatibility](compatibility.md#guest-distribution-qualification) owns qualification evidence
and requalification.

## VM DNS search domain

[ADR 0019](decisions/0019-explicit-vm-dns-search-domain.md) requires an explicit consumer-owned
search domain as a breaking addition to the existing VM bootstrap interface. It is approved
Architecture for a later focused implementation, not a claim that the input is already
available.

The contract belongs in the existing VM module and resource. Its required input, fictional
domain example, breaking adoption guidance and cloud-init/FQDN/reboot warnings belong in the
module README and existing consumer guidance when implemented. That implementation keeps the
existing consumer example and validation fixtures consistent with the new required input; it
introduces no new example, module, Ansible role, DNS abstraction or orchestration, and leaves
the separate LXC contract unchanged.

The input is `dns_search_domain`: a required, non-null string with no default, containing one
ASCII DNS name. Labels are 1–63 letters/digits or inner hyphens, separated by dots; the complete
value is at most 253 characters and has no trailing dot. A single-label domain is allowed. The
existing VM `name` remains one DNS label of at most 63 characters; the generated `name.domain`
must also fit 253 characters, so a known overlong combination is rejected before apply even when
each input is valid alone. These presentation limits preserve the [DNS label and encoded-name
bounds](https://www.rfc-editor.org/rfc/rfc1035.html#section-2.3.4) for this ASCII interface.
Null, empty or whitespace-containing values, domain lists, malformed names and the bare `.`
value are rejected before apply. The validated value maps unchanged to
`initialization.dns.domain`; it is not derived from the VM name, template or node. No no-search
mode or sentinel is exposed.

DNS servers remain required and unchanged, including the existing IPv4/IPv6 server contract. The
domain is global cloud-init DNS configuration in both primary IPv4 modes and with optional
static IPv6; it adds no per-attachment addressing. Primary/additional network mapping, gateway,
MAC and VLAN behavior, and the required `connection` shape and derivation remain unchanged. No
domain is automatically appended to connection metadata.

OpenTofu owns the Proxmox cloud-init domain declaration. Consumers own the domain, DNS service
and namespace, suitable template and consistency with the existing VM name and environment.
Ongoing guest resolver configuration remains an Ansible/consumer concern. There is no
DNS-service, DHCP-service, IPAM, guest-network, secret or orchestration mechanism.

The pinned VM-specific source shows two inheritance paths: clone copies an explicit template
domain; absent a VM domain, Proxmox's cloud-init rendering independently uses the selected
node's search domain even when nameservers are explicit. The provider sends and reads back a
declared nonempty value. Empty instead omits the request field and does not clear a previously
configured VM domain; read-back can retain it and produce recurring differences. This is not
LXC's update/deletion path, and repeated apply, ignored differences or direct Proxmox cleanup
are not a supported remedy.

Initial declaration, adoption on an existing VM, and changing one valid nonempty domain to
another must preserve the VM and reach a subsequent no-change plan grounded in the provider
update/read paths. Removing the declaration, empty values, return to inheritance and no-search
operation are unsupported. A provider change causing replacement or recurring differences for a
supported transition is blocked pending Architecture review.

Adding or changing the domain declaration can change generated user/network data. Proxmox
generates `name.domain` as the FQDN for the module's existing single-label VM name. Explicitly
choosing the same value as the previous node fallback can still change the generated FQDN: the
hostname helper does not use that fallback. The upstream helper also retains an already
qualified name, but that inspected branch does not add qualified-name support to this module.
The pinned initialization update path requests cloud-init rebuild and reboot for a running VM,
with changed NoCloud instance identity when rendered data changes. Consumers must expect the
existing reconfiguration, possible SSH host-key regeneration and connectivity-loss consequences,
not assume a cosmetic DNS-only change or zero downtime. These are pinned source expectations,
not live observations.

Before Developer acceptance, credential-free module tests must prove
required/null/empty/name/list validation, accepted and rejected generated-FQDN length
boundaries, pass-through mapping, unchanged server/addressing/connection contracts and absence
of a no-search/default mode. Separate exact-locked-provider evidence must cover first explicit
initialization, adoption from node fallback and an explicit inherited-template domain, and one
nonempty-domain change; preserve the VM and unrelated networking; and show a subsequent
no-change plan for each update using source-grounded read-back. The record distinguishes
executed plans, modeled read-back and inspected create/update/read/rendering paths from live
evidence not obtained. Provider upgrades re-evaluate the same gate. The implementation must also
keep the retained module consumption minimum, with the consuming-root evidence [ADR
0016](decisions/0016-module-cli-minimum-and-evidence-policy.md) requires; the combined name and
domain rule spans two inputs, so it cannot rely on a newer CLI's cross-variable validation.

Architecture acceptance does not require finished module tests. Implementation must separately
reconcile the existing module README, consumer guidance and existing example/fixtures with the
breaking required input, using fictional domains; it creates no new example or consumption model
and performs no automatic consumer migration. Public checks do not establish Proxmox
apply/refresh, full cloud-init execution, guest resolver/hostname changes, DNS resolution,
host-key behavior, reboot success or reachability. Explicit no-search support remains deferred
until an independently reviewed path establishes its semantics and convergence.
