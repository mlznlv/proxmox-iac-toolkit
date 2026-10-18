# Compatibility

This page owns targets and evidence. At a release commit it is that release's snapshot; `main`
can include unreleased changes. Executable `versions.tf` files own module CLI/provider
constraints; contributor versions belong to [the declarations](../CONTRIBUTING.md#set-up).
[Release policy](release-policy.md#maturity) owns maturity and support commitments.

## Support and evidence matrix

| Surface | Target | Evidence and limit |
| --- | --- | --- |
| [VM](../tofu/modules/proxmox-linux-vm/README.md) | Proxmox VE 9.x; consumer cloud-init template meeting its README contract | Static/mocked declarations and exact-provider lifecycle evidence; no distribution-specific VM template qualification |
| [LXC](../tofu/modules/proxmox-linux-container/README.md) | Proxmox VE 9.x; Debian Stable template meeting its README contract | Expected-compatible; mocked declarations, provider plans and pinned setup source |
| [Guest-agent role](../ansible/roles/qemu_guest_agent/README.md) | Debian Stable and Kali Rolling meeting its capability contract | Expected-compatible; syntax/lint/static role checks and packaging inspection |
| [Consumer example](../examples/separate-consumer-repository/README.md) | One pinned checkout for VM and role | Structural resolution and inventory mapping only; adds no platform target |
| [Live integration](#live-integration-evidence) | None recorded | No runtime-validated reference platform |

Existing release-series targets retain their prior wording/evidence until their next
requalification trigger; they do not cover an unevaluated release. Other Debian-family guests
may meet a role contract without gaining a toolkit claim. The supported PVE major does not
float: adopting another requires deliberate compatibility review, validation and declaration
updates; a changed durable policy requires an ADR.

## OpenTofu CLI compatibility and evidence

[ADR 0016](decisions/0016-module-cli-minimum-and-evidence-policy.md) retains 1.6.0 as the
consumer minimum for both modules.

**The VM source with trimmed SSH keys meets its declared minimum in the checks run.** The VM
module of [#134](https://github.com/mlznlv/proxmox-iac-toolkit/pull/134), which adds the key
change of [#133](https://github.com/mlznlv/proxmox-iac-toolkit/issues/133) to the required
search domain, and whose `tofu/modules/proxmox-linux-vm` directory is Git tree
`cc32485145e7af26ae3f82e5bb83ebd8eb1edeec`, was checked unmodified through a fictional consuming
root on 1.6.0 and 1.9.4, and on the contributor pin, with the module's committed provider lock
read only. It passed the cases of #123's and #135's records below, with the search domain
declared and the planned keys checked too, and a case with whitespace around each key, which
planned each key trimmed. [The evidence
record](https://github.com/mlznlv/proxmox-iac-toolkit/pull/134#issuecomment-6250258845) gives
every case and its result, and names the earlier record that gives the CLI and provider
identities. These are sampled initialization, validation and plan results, not full feature
qualification or an interval-wide guarantee.

**The VM source with the required search domain meets its declared minimum in the checks run.**
The VM module of [#135](https://github.com/mlznlv/proxmox-iac-toolkit/pull/135), whose
`tofu/modules/proxmox-linux-vm` directory is Git tree
`ca015a928287aee35feba0c1ab559373da9a74bb`, was checked unmodified through a fictional consuming
root on 1.6.0 and 1.9.4, and on the contributor pin, with the module's committed provider lock
read only. Initialization and validation succeeded with every nullable input at its default.
Single- and multi-label search domains, DHCP, and static IPv6 beside an additional attachment
planned with refresh disabled, each with the domain unchanged, and so did a 63-character name
with a 189-character domain, 253 characters together. An omitted domain stopped with OpenTofu's
required-argument error and a list with its type error; null, empty, whitespace, malformed,
overlong, trailing-dot and bare-dot values and a qualified name each stopped with the module's
own message; and a name and domain of 254 characters together stopped at the module's
precondition. [The evidence
record](https://github.com/mlznlv/proxmox-iac-toolkit/pull/135#issuecomment-6244971461) gives
the CLI and provider identities, every case and its result. These are sampled initialization,
validation and plan results, not full feature qualification or an interval-wide guarantee.

The corrected VM source in [PR #123](https://github.com/mlznlv/proxmox-iac-toolkit/pull/123),
module tree `13e5f2b8b576d4a501c5bb7048e873b53831ec68`, passed consuming-root checks on 1.6.0
and 1.9.4 with unmodified source, fictional inputs and its committed provider lock read-only.

| Corrected VM check | Recorded result on both CLIs |
| --- | --- |
| Initialization and validation with nullable defaults | Passed, including `vm_id` and `network_vlan_id` |
| Address, gateway, DNS and SSH-key rules | Accepted valid boundaries; rejected #112 cases and adjacent octet/prefix failures with module messages |
| Refresh-disabled plans against fictional unreachable endpoint | DHCP, static IPv6, mixed-family DNS, additional attachment and connection host passed; static/DHCP and duplicate-address preconditions rejected invalid cases |

The [correction evidence](https://github.com/mlznlv/proxmox-iac-toolkit/pull/123#issuecomment-6236986003)
records CLI/provider identities and each case. Earlier unmodified source
`3a8553a16bcfd5a4e78957988d96f0b5fc9ab543` had these nullable-default init/validate results:

| CLI | VM | LXC |
| --- | --- | --- |
| 1.6.0, 1.9.4 | Failed identifier/VLAN guards | Passed baseline |
| 1.10.0, 1.12.6 | Passed baseline | Passed baseline |

The [baseline
investigation](https://github.com/mlznlv/proxmox-iac-toolkit/issues/120#issuecomment-6233199326)
distinguishes unchanged-source checks from scratch-patched diagnostics. All results are sampled
behavior at recorded identities, not full-feature or interval-wide qualification.
[Contributing](../CONTRIBUTING.md#module-consumption-compatibility-evidence) owns the procedure;
contributor-pin mocks do not establish the minimum or provider/guest runtime behavior. The
expired provider signing key of [#111](https://github.com/mlznlv/proxmox-iac-toolkit/issues/111)
was fixed upstream by renewing the same key, so the locked v0.111.1 now installs without the
expired-key warning with the committed locks unchanged; [the remediation
record](https://github.com/mlznlv/proxmox-iac-toolkit/issues/111#issuecomment-6233138082) gives
the signing provenance and acquisition runs.

## Provider lifecycle evidence

The reviewed build is `bpg/proxmox` v0.111.1, source `b22fe919fc34476b232191b69c8907f0c1aa5bea`.
The provider constraint is deliberately narrow: `bpg/proxmox` is pre-1.0, so a minor release may
change behavior, and adopting one is a compatibility decision rather than an automatic upgrade.
Repository locks pin validation; consumers select their own root locks. A merely allowed build
acquires no evidence. The component READMEs own supported transitions and disruption/destruction
rules.

| Capability | Evidence source | What was established |
| --- | --- | --- |
| Agent channel | [ADR 0006](decisions/0006-guest-agent-channel-at-creation.md) | Pinned Proxmox/packaging/provider source; channel/wait separation |
| VLAN | [ADR 0009](decisions/0009-first-m6-vlan-expansion.md), VM README history | Schema/source and refresh-disabled exact-build plans for add/change/remove |
| Additional attachments | [PR #102](https://github.com/mlznlv/proxmox-iac-toolkit/pull/102), [ADR 0014](decisions/0014-additional-attachment-retirement-lifecycle.md) | Synthetic-state plans, source-grounded modeled read-back, supported updates and retained-entry counterexamples |
| DHCP/primary MAC | [PR #114](https://github.com/mlznlv/proxmox-iac-toolkit/pull/114), [ADR 0012](decisions/0012-dhcp-primary-attachment-addressing.md) | Mode/MAC mapping, in-place plans and modeled subsequent no-change plans |
| Static IPv6 | [PR #113](https://github.com/mlznlv/proxmox-iac-toolkit/pull/113), [ADR 0013](decisions/0013-static-ipv6-primary-attachment.md) | Add/change/remove plans and modeled convergence across IPv4 modes; pinned cloud-init renderer source |
| VM DNS search domain | [PR #135](https://github.com/mlznlv/proxmox-iac-toolkit/pull/135), [ADR 0019](decisions/0019-explicit-vm-dns-search-domain.md) | Declare, adopt and change plans in both IPv4 modes and beside IPv6 and an additional attachment, modeled subsequent no-change plans; pinned provider and qemu-server source |
| VM bootstrap SSH keys | [PR #134](https://github.com/mlznlv/proxmox-iac-toolkit/pull/134), [#133](https://github.com/mlznlv/proxmox-iac-toolkit/issues/133) | Send/read-back source and plans against modeled read-back: one keys-only in-place update where needed, then no change |
| LXC | [PR #107](https://github.com/mlznlv/proxmox-iac-toolkit/pull/107), [ADR 0015](decisions/0015-first-lxc-lifecycle-and-dns-contract.md) | Synthetic-state lifecycle plans plus provider/SDK/Proxmox inspection for keys, DNS, shrink, nesting, reboot and read/destroy behavior |

No apply or live refresh was executed by this evidence. Source-grounded modeled read-back is
distinct from an observed no-op after apply. Provider upgrades must re-evaluate exact-build
planning and update/read paths under the [acceptance
procedure](../CONTRIBUTING.md#component-evidence-requirements). Unexpected replacement or
recurring differences for a supported update block adoption pending Architecture review.

### VM DNS search domain

[ADR 0019](decisions/0019-explicit-vm-dns-search-domain.md) adopts a required explicit
consumer-owned single DNS search domain for the VM. This is a breaking input addition,
implemented as the module's required `dns_search_domain` input by
[#135](https://github.com/mlznlv/proxmox-iac-toolkit/pull/135), not a new PVE/provider target.
The VM name remains one DNS label; the combined generated `name.domain` must pass the locally
decidable length limit, not just each input separately. DNS servers remain required, and primary
IPv4 static/DHCP, optional static IPv6, additional attachments and declared `connection`
metadata retain their contracts.

VM-specific source for locked v0.111.1 at `b22fe919fc34476b232191b69c8907f0c1aa5bea` and
qemu-server at `14c0f871fa926e04eee090afae5d2d8670aa4a79` establishes independent template/node
search-domain inheritance despite explicit nameservers, nonempty-domain request/read mapping and
the absent empty-domain deletion path. The VM API accepts a string; neither that acceptance nor
LXC's different schema establishes a safe VM no-search sentinel.

Supported explicit-domain adoption and nonempty-to-nonempty changes must preserve the VM and
converge. Developer acceptance requires exact-provider implementation evidence and subsequent
no-change plans grounded in update/read behavior, separately from module mocks. Removal and
return to inheritance are unsupported. A build merely allowed by the provider constraint
acquires no lifecycle evidence until reviewed; upgrades must re-evaluate this gate.

The implementation's [exact-provider
record](https://github.com/mlznlv/proxmox-iac-toolkit/pull/135#issuecomment-6244965948) used the
locked v0.111.1 build, refresh disabled, and synthetic prior states, in static and DHCP modes
and beside static IPv6 and an additional attachment. Creating a VM with a declared domain
planned a create. Declaring one on a VM without its own, replacing one its template supplied,
and changing one declared domain to another each planned an in-place update with no replacement
path that changed only the search domain, and a further plan against a state modelled on the
provider's read path proposed nothing. Declaring the template's own domain planned no change.
This is provider planning plus source evidence, not executed apply/refresh or live guest
evidence.

The [investigation
record](https://github.com/mlznlv/proxmox-iac-toolkit/issues/121#issuecomment-6233199488)
distinguishes pinned source inspection from extracted DNS/hostname-helper probes with a
fictional resolver reader. These are not full installed Proxmox/cloud-init or guest tests.
Initialization updates can rebuild cloud-init and reboot a running VM; generated FQDN and
instance identity may change, with the established reconfiguration, possible host-key
regeneration and connectivity consequences. None of this establishes actual guest DNS, hostname,
reboot, SSH or reachability behavior. No provider upgrade, LXC DNS change or live-testing
requirement is introduced.

### VM bootstrap SSH keys

The locked v0.111.1 at `b22fe919fc34476b232191b69c8907f0c1aa5bea` joins the bootstrap keys with
newlines when it sends them and, when it reads them back, trims whitespace from the ends of the
joined value only and splits it on newlines, with no diff suppression; qemu-server at
`14c0f871fa926e04eee090afae5d2d8670aa4a79` stores `sshkeys` as sent. Whitespace before the first
key or after the last, or a newline ending any other, therefore read back changed and planned an
in-place `initialization` update on every run, which by the provider's source rebuilds the
cloud-init drive and reboots a running VM. Since
[#133](https://github.com/mlznlv/proxmox-iac-toolkit/issues/133), the module sends each key
trimmed.

[The
reproduction](https://github.com/mlznlv/proxmox-iac-toolkit/issues/133#issuecomment-6241658338)
and the [adoption
record](https://github.com/mlznlv/proxmox-iac-toolkit/pull/134#issuecomment-6250238133) on the
source that also requires the search domain are plans on 1.12.6 and 1.6.0 with refresh disabled,
against prior states written through a model of the provider's send and read paths and of
Proxmox storing the keys as sent. A VM created by the corrected module plans no change. An
existing VM plans no change when the provider's read of its keys already equals each declared
key trimmed. Otherwise it plans one in-place update that changes only `keys`, including where
several keys planned no change before, and a further plan against that update's modelled
read-back proposes nothing. The [module
README](../tofu/modules/proxmox-linux-vm/README.md#bootstrap-ssh-keys) and the
[changelog](../CHANGELOG.md) say which configurations are affected.

Neither record observes Proxmox storing or returning the keys, a refresh, an apply, a reboot, or
cloud-init on a guest. That removing only an empty key can leave the generated user data
unchanged, while removing other whitespace can change it and with it the cloud-init instance, is
read from qemu-server's source. Whether a guest then regenerates its SSH host keys depends on
its template, and none of this establishes guest SSH access or reachability.

## Current evidence level

| Class | Proves | Does not prove |
| --- | --- | --- |
| Static/mocked | Declared inputs, validation, provider fields, outputs, role tasks and example composition | Provider lifecycle, authorization or a running guest |
| Exact-provider | Executed plans at recorded CLI/build/source; inspected update/read behavior | Actual apply, refresh, hot-plug, MAC continuity or guest convergence |
| Pinned guest source | Packaging, cloud-init and setup-plugin expectations at inspected versions | Template behavior observed on a guest |
| Guest runtime | Only recorded scenarios at exact recorded identities | Other versions/environments or untested operations; none exists here |

No public check establishes live provisioning, clone/start/update/destroy,
shutdown/reboot/backup, cloud-init, host-key behavior, SSH/privilege escalation, agent
operation, Ansible convergence/idempotency, VLAN/routing, DHCP lease or naming, IPv6/RA, LXC
setup/nesting/DNS, checkout acquisition, a consumer's connectivity or compatibility with a live
consumer environment. Static/mocked evidence must never be called runtime validation.

A distribution or platform becomes a runtime-validated reference platform only after a [live
run](#live-integration-evidence) exercises it to the level claimed and this contract records
that evidence, and then only for the identities recorded; for a guest distribution,
[Guest-distribution qualification](#guest-distribution-qualification) also applies.

## Live integration evidence

None exists. Every claim on this page rests on static, mocked, exact-provider, or source
evidence, and nothing has run against a PVE installation or a guest.

[ADR 0018](decisions/0018-isolated-live-validation-path.md) defines the only path through which
the toolkit collects live evidence: opt-in, run by an operator outside normal public CI, against
an isolated reference environment, for a revision already merged to `main`. A run recorded here
is runtime evidence for exactly its scenarios at the identities its sanitized record names: the
toolkit commit; the OpenTofu, provider build, `ansible-core`, and controller Python versions;
the PVE version and the Proxmox packages on the path; whether the installation is nested; each
template's public identity and checksum; and each guest's distribution, architecture, release,
and cloud-init, init system, and Python versions. It sits beside the evidence above and replaces
none of it, and every run of the same scenarios at the same identities is reported, failed and
inconclusive runs included.

A recorded run establishes nothing for another PVE 9.x version, another provider build or tool
version, another template or distribution, another storage or network configuration, or a
consumer's environment, and nothing about reliability across runs, performance, availability,
uninterrupted operation, scale, or any transition it did not exercise. Unsupported transitions
are never recorded as supported. A claim stays attached to its recorded identities until a run
at a changed identity is recorded.

## Guest-distribution qualification

[ADR 0017](decisions/0017-guest-distribution-extension-and-qualification.md) owns why tracks and
expansion gates are separate. **Not targeted** is the default; **expected-compatible** requires
applicable static/mocked, exact-provider and pinned guest-source evidence; **runtime-validated**
requires recorded guest execution through an accepted isolated live path. Consumer runs may
demonstrate need but are not toolkit runtime evidence. Support does not transfer between VM
templates, LXC, individual roles or their composition.

| Track | Required qualification inputs |
| --- | --- |
| VM | Template provenance; cloud-init version/datasource/format/renderer and covered capabilities; default-user privilege; first-boot/package upgrades; new-instance/network/host-key effects; agent packaging when paired with a role. Provider evidence is refreshed when module/provider changes. |
| LXC | Type/detected distribution agreement, pinned setup plugin and release limits; creation/every-start network/hostname/DNS/root-key behavior; unprivileged init/nesting and root SSH template contract; provider create/type-read/template-and-type lifecycle evidence. |
| Each role | Python, package manager/name, service manager/unit/activation, non-interactive privilege, inventory/connection and device requirements; runtime claims require convergence and a no-change second run. |

Each sanitized record identifies component/track, distribution/architecture, exact release
(dated image/snapshot for rolling releases), public template/image identity and published
checksum, PVE major/relevant packages, provider build/source, OpenTofu, Ansible, cloud-init,
init-system and Python versions as applicable, toolkit commit, evidence classes, executed versus
inspected work, failures and counterexamples. Never use a private volume/storage path as image
identity.

Requalify affected tracks before retaining claims when release/series or end of life changes; an
image rebuild changes cloud-init/init/packages/units/SSH defaults; PVE major/relevant package or
provider changes; Ansible changes Python/module support; the component contract changes; or a
credible counterexample appears. Retained claims need an Architecture-reviewed record update.
Narrowing/withdrawal also needs changelog and migration guidance; existing guests are not
converted.
