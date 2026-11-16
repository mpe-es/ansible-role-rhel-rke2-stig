# Ansible Role: RHEL RKE2 STIG Compliance

[![CI](https://github.com/darkhonor/ansible-role-rhel-rke2-stig/actions/workflows/ci.yml/badge.svg)](https://github.com/darkhonor/ansible-role-rhel-rke2-stig/actions/workflows/ci.yml)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-darkhonor.rhel__rke2__stig-blue.svg)](https://galaxy.ansible.com/darkhonor/rhel_rke2_stig)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![RHEL 9 STIG](https://img.shields.io/badge/RHEL%209%20STIG-V2R8-green.svg)](https://public.cyber.mil/stigs/)
[![RHEL 10 STIG](https://img.shields.io/badge/RHEL%2010%20STIG-V1R1-green.svg)](https://public.cyber.mil/stigs/)
[![RKE2 STIG](https://img.shields.io/badge/RKE2%20STIG-V2R6-green.svg)](https://public.cyber.mil/stigs/)
[![Container SRG](https://img.shields.io/badge/Container%20SRG-V2R4-green.svg)](https://public.cyber.mil/stigs/)

Supplementary Ansible role for enforcing DISA STIG compliance on RHEL 9/10
systems running RKE2 or K3S Kubernetes distributions.

**Current STIG Baselines:**
- RHEL 9 V2R8 (Released: 01 Apr 2026)
- RHEL 10 V1R1 (initial release)
- RKE2 V2R6 (Released: 01 Apr 2026)
- Container Platform SRG V2R4 (Released: 28 Oct 2025)

The role auto-detects the target's RHEL-family major version and applies the
matching OS baseline (RHEL 9 V2R8 or RHEL 10 V1R1). See
[RHEL 10 STIG Support](#rhel-10-stig-support) for the version-gating model and
the RHEL 9 to RHEL 10 differences.

## Overview

This role is designed to run **after** the operating system is installed with a
STIG profile applied (e.g., via RHEL's OSCAP Anaconda addon). It fills in gaps
and adds kernel/system-level configurations required for Kubernetes nodes in
DoD environments.

**This role does NOT implement the full RHEL STIG.** It assumes the base STIG
profile was applied at install time and handles:

- Missing or incomplete STIG settings post-installation
- Audit rules for system call monitoring and privileged commands
- Kernel module blacklisting for attack surface reduction
- System tuning common to RKE2/K3S Kubernetes nodes

## Requirements

- **Ansible:** 2.10 or higher
- **Python:** 3.9 or higher on control node
- **Target OS:** RHEL 9 or RHEL 10 (Enterprise Linux family)
- **Privileges:** Root access required on target hosts
- **Pre-requisite:** STIG profile applied during OS installation

### Ansible Collections

This role requires the following collections (installed automatically if using
`ansible-galaxy`):

```yaml
collections:
  - ansible.posix
  - community.general
```

## Role Variables

### Version-Specific Static Data (vars/rhel9.yml, vars/rhel10.yml)

The role loads the audit-rule and kernel-blacklist data set for the detected
RHEL-family major version (`include_vars vars/rhel<major>.yml`). These define
STIG-required configurations and are not typically modified:

| Variable | Description |
|----------|-------------|
| `rhel_rke2_stig_audit_rules` | Audit rule definitions with STIG metadata (RHEL 9 and RHEL 10 each ship 50 rules, mapped to their version's STIG IDs) |
| `rhel_rke2_stig_blacklist_modules` | List of 8 kernel modules to blacklist |

`vars/main.yml` holds only the shared constant `rhel_rke2_stig_supported_majors`
(`["9", "10"]`).

### Default Variables (defaults/main.yml)

These tunables enable/disable controls or set their values. The full list with
inline documentation is in `defaults/main.yml`; the most commonly adjusted:

| Variable | Default | Purpose |
|----------|---------|---------|
| `rhel_rke2_stig_crypto_policy` | `FIPS:AD-SUPPORT` | System-wide crypto policy for AD/IdP integration (see [RHEL-09-215105](#rhel-09-215105-fips-cryptographic-policy)) |
| `rhel_rke2_stig_rp_filter_all` | `"2"` | RHEL 10 only: `net.ipv4.conf.all.rp_filter` (loose by default; STIG strict is `"1"`) |
| `rhel_rke2_stig_ipv6_enabled` | `false` | RHEL 10 only: enable IPv6 forwarding as a dual-stack RKE2 exemption |
| `rhel_rke2_stig_smartcard_enabled` | `false` | CAC smart card driver (RHEL-09-611160) |
| `rhel_rke2_stig_sssd_enabled` | `false` | SSSD certificate mapping / cached-credential expiry |
| `rhel_rke2_stig_orphan_files_enabled` | `false` | Orphaned-file remediation (off; can break applications) |
| `rhel_rke2_stig_aide_exclusions_enabled` | `true` | Exclude Kubernetes container logs from AIDE (see [AIDE and container logs](#aide-and-container-logs)) |
| `rhel_rke2_stig_aide_exclusions` | `[/var/log/pods, /var/log/containers]` | Paths excluded from AIDE file integrity checks |
| `rhel_rke2_stig_audit_immutable_pending_is_fatal` | `true` | Fail when audit rule changes cannot be applied because audit is already immutable (`-e 2`) |
| `rhel_rke2_stig_<area>_enabled` | `true` | Per-area on/off toggles (ssh, pam, fapolicyd, postfix, file_permissions, ...) |

### AIDE and container logs

Stock `/etc/aide.conf` watches `/var/log` recursively. `kubelet` creates
`/var/log/pods/` and `/var/log/containers/` when RKE2 installs — after the image is
baked — so they are swept into that rule silently. The SSG remediation
`aide_use_fips_hashes` then adds `sha512` to the `LOG` group, which is meant to be
size-only, so those append-only container logs are **content hashed**. On a running
node they change every few seconds and `aide --check` can never return `0`.

A file integrity check that is permanently red is worse than none: a real intrusion
is indistinguishable from the expected noise. This role therefore excludes the
Kubernetes paths by default.

`/var/log/audit/audit.log` has the same churn from the same cause but is
deliberately **not** excluded — it is a base-OS concern on every RHEL host rather
than an RKE2 gap, and excluding it would weaken a base control this role does not
own.

Changing these exclusions requires re-initializing the AIDE database
(`aide --init && mv -f /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz`).
Entries already in the database for now-excluded paths otherwise report as
`REMOVED` on the next check. The role warns but does not do this: a valid baseline
must capture the final state of a specific deployed node, which is not knowable at
role-apply or image-bake time.

### Expected SCAP failures: cron permissions

An OpenSCAP run with the STIG profile reports `file_permissions_cron_d`,
`_daily`, `_hourly`, `_weekly`, `_monthly` and `file_permissions_crontab` as
**failed** on a compliant node. That is correct and must not be "fixed".

DISA `RHEL-09-232040` (V-257888) requires that cron permissions *"not be modified
from the operating system defaults"*. The packaged defaults are `0755` on the cron
directories and `0644` on `/etc/crontab` (`rpm -q --dump cronie`), and this role's
`rpm --setperms` restores exactly that.

SSG's `file_permissions_cron_*` rules demand `0700`/`0600`, but in
`scap-security-guide` 0.1.81 they carry **no `RHEL-10-nnnnnn` DISA rule ID** — their
only DISA reference is the generic catch-all `SRG-OS-000480-GPOS-00227`, alongside
CIS and NIST refs — and the RHEL 10 datastream contains no STIG-ID-mapped cron
permission control at all. SSG is applying CIS-derived hardening stricter than any
specific DISA cron requirement, and on RHEL 9 it directly contradicts the authored
control. The STIG is the compliance authority; the scanner profile is not.

## RHEL 10 STIG Support

This role supports both RHEL 9 (DISA STIG V2R8) and RHEL 10 (DISA STIG V1R1),
selected automatically from the target's `ansible_distribution_major_version`:

- `tasks/main.yml` asserts the host is RHEL-family and a supported major (`9` or
  `10`), then loads `vars/rhel<major>.yml` (the version's audit rules + blacklist).
- The version-varying audit-rule and blacklist data is selected by the
  `include_vars` above. Controls that are entirely new to RHEL 10 (the
  `all.rp_filter` and IPv6-forwarding sysctls) are additionally gated with
  `when: ansible_distribution_major_version is version('10', '>=')`.
- Per-control task tags use the RHEL 9 STIG IDs as a stable namespace across both
  versions (e.g. `--tags RHEL-09-611075`); on RHEL 10 the task applies the mapped
  RHEL 10 control. Area tags (`audit`, `kernel`, `ssh`, `sysctl`, `rke2`, ...) and
  full-role runs work on all supported versions. RHEL 10 STIG IDs are not tag
  selectors.
- RHEL 9 behavior is unchanged by the addition of RHEL 10 support.

### What differs on RHEL 10 (V1R1)

| Area | RHEL 9 (V2R8) | RHEL 10 (V1R1) |
|------|---------------|----------------|
| Audit rule IDs | RHEL-09-654010 - 654255 | RHEL-10-500300 - 500810 (50 rules) |
| Identity-file audit | `-w <path> -p wa` watch form | `-F path=<path> -F perm=wa` syscall pairs (b32/b64) |
| chmod audit | `chmod,fchmod,fchmodat` | adds `fchmodat2` syscall |
| rename/delete audit | `rename,...,unlinkat` | adds `renameat2` syscall |
| `all.rp_filter` | not required | RHEL-10-800130 (applied permissively; see below) |
| IPv6 forwarding | n/a | RHEL-10-800250 (toggle; STIG-enforced off by default) |
| User namespaces | RHEL-09-213105 exemption | control retired in V1R1; setting kept operationally |
| Kernel blacklist | 8 modules | same 8 (atm/firewire-core/cramfs retired in V1R1 but kept as defense-in-depth) |

### RHEL 10 operational notes

- **`all.rp_filter` is applied permissively** (`rhel_rke2_stig_rp_filter_all`,
  default loose `"2"`) rather than STIG-strict (`"1"`), because strict
  reverse-path filtering can drop asymmetrically-routed RKE2/CNI traffic. Set to
  `"1"` after validating on your cluster.
- **IPv6 forwarding is disabled by default** (STIG-compliant). Set
  `rhel_rke2_stig_ipv6_enabled: true` to enable it as a documented dual-stack
  exemption (mirrors the IPv4 forwarding exemption).

### Corrected DISA V1R1 source defects

Two RHEL 10 V1R1 audit fix-text entries specify rules the audit kernel rejects on
load; the role ships the corrected, loadable form (which still satisfies each
rule's own STIG check text) with the deviation documented inline:

- **RHEL-10-500720** (`/etc/opasswd`): fix text has a stray `(` before the b64
  path. Tracked in issue #21.
- **RHEL-10-500690** (`/etc/sudoers.d/`): fix text uses a trailing-slash directory
  path that yields `EINVAL` on load. Tracked in issue #22.

## STIG Coverage

The STIG IDs below are shown with their **RHEL 9 (V2R8)** identifiers. On RHEL 10
the role applies the equivalent **RHEL 10 (V1R1)** controls automatically; see
[RHEL 10 STIG Support](#rhel-10-stig-support) for the ID mapping and differences.

This role currently addresses **81 STIG findings**:

### Cryptographic Policy (1 finding)

| STIG ID | Severity | Description |
|---------|----------|-------------|
| RHEL-09-215105 | CAT I | FIPS 140-3 compliant cryptographic policy (configurable) |

### Password Policy (4 findings)

| STIG ID | Severity | Description |
|---------|----------|-------------|
| RHEL-09-411010 | CAT II | Maximum password lifetime (60 days) |
| RHEL-09-611050 | CAT II | PAM password-auth SHA512 hashing rounds (configurable) |
| RHEL-09-611055 | CAT II | PAM system-auth SHA512 hashing rounds (configurable) |
| RHEL-09-611075 | CAT II | Minimum password lifetime (24 hours) |

### SSH Configuration (5 findings)

| STIG ID | Severity | Description |
|---------|----------|-------------|
| RHEL-09-255035 | CAT II | SSH public key authentication enabled |
| RHEL-09-255070 | CAT I | SSH client MACs (FIPS 140-3 validated algorithms) |
| RHEL-09-255075 | CAT I | SSH server MACs (FIPS 140-3 validated algorithms) |
| RHEL-09-255095 | CAT II | SSH ClientAliveCountMax (terminate unresponsive connections) |
| RHEL-09-255100 | CAT II | SSH ClientAliveInterval (10 minute max idle timeout) |

### NetworkManager Configuration (1 finding)

| STIG ID | Severity | Description |
|---------|----------|-------------|
| RHEL-09-252040 | CAT II | Configure DNS processing mode in NetworkManager |

### Smart Card / PKI Configuration (3 findings)

| STIG ID | Severity | Description |
|---------|----------|-------------|
| RHEL-09-611160 | CAT II | CAC smart card driver configuration (disabled by default) |
| RHEL-09-631015 | CAT II | SSSD certificate-to-user mapping (disabled by default) |
| RHEL-09-631020 | CAT II | SSSD offline credentials expiration (1 day) |

> **Note:** Smart Card and SSSD features are disabled by default as they require
> environment-specific configuration. Enable via `rhel_rke2_stig_smartcard_enabled`
> and `rhel_rke2_stig_sssd_enabled` variables. See [Role Variables](#role-variables).

### File Permissions (5 findings)

| STIG ID | Severity | Description |
|---------|----------|-------------|
| RHEL-09-232040 | CAT II | Reset cron configuration permissions to RPM defaults |
| RHEL-09-232045 | CAT II | Local initialization files mode 0740 (disabled by default) |
| RHEL-09-232245 | CAT II | Set sticky bit on world-writable directories |
| RHEL-09-232250 | CAT II | Ensure files have valid group owner (disabled by default) |
| RHEL-09-232255 | CAT II | Ensure files have valid owner (disabled by default) |

> **Note:** RHEL-09-232045 (init files), RHEL-09-232250, and RHEL-09-232255 (orphaned
> files) are disabled by default as auto-remediation can break user environments or
> applications. Enable after manual review of your environment.
>
> **RKE2/K3S Exclusion:** The orphaned file checks automatically exclude the path
> defined in `rhel_rke2_stig_rancher_base_path` (default: `/var/lib/rancher`) because
> container files use UIDs/GIDs that don't map to host users. See
> [STIG Exemptions](#rhel-09-232250--rhel-09-232255-orphaned-files-in-container-paths)
> for assessor documentation.

### Mail Configuration (1 finding)

| STIG ID | Severity | Description |
|---------|----------|-------------|
| RHEL-09-252050 | CAT II | Configure postfix to prevent unrestricted mail relaying |

### Audit Rules (50 findings)

| STIG ID Range | Severity | Description |
|---------------|----------|-------------|
| RHEL-09-654010 - 654080 | CAT II | System call auditing (execve, chmod, chown, etc.) |
| RHEL-09-654085 - 654175 | CAT II | Privileged command auditing (passwd, sudo, su, etc.) |
| RHEL-09-654180 - 654210 | CAT II | Mount/system control auditing |
| RHEL-09-654215 - 654255 | CAT II | Identity file change auditing |

### Kernel Hardening (3 findings)

| STIG ID | Severity | Description |
|---------|----------|-------------|
| RHEL-09-213040 | CAT II | Disable core dumps via kernel.core_pattern |
| RHEL-09-213080 | CAT II | Restrict ptrace to descendant processes only |
| RHEL-09-253050 | CAT II | Enable reverse-path filter for IPv4 by default |

### Kernel Module Blacklist (8 findings)

| STIG ID | Severity | Module | Description |
|---------|----------|--------|-------------|
| RHEL-09-213045 | CAT II | atm | Asynchronous Transfer Mode |
| RHEL-09-213050 | CAT II | can | Controller Area Network |
| RHEL-09-213055 | CAT II | firewire-core | FireWire |
| RHEL-09-213060 | CAT II | sctp | Stream Control Transmission Protocol |
| RHEL-09-213065 | CAT II | tipc | Transparent Inter Process Communication |
| RHEL-09-231195 | CAT III | cramfs | Compressed ROM filesystem |
| RHEL-09-291010 | CAT II | usb-storage | USB mass storage |
| RHEL-09-291035 | CAT II | bluetooth | Bluetooth |

### RKE2 Operational Requirements

This role also deploys RKE2-specific configurations that are not direct STIG
requirements but are necessary for Kubernetes operation:

| Configuration | File Deployed | Purpose |
|---------------|---------------|---------|
| Kernel Modules | `/etc/modules-load.d/rke2-modules.conf` | Load networking modules for CNI |
| Sysctl Tuning | `/etc/sysctl.d/98-rke2.conf` | Kernel parameters for K8s networking |
| STIG Override | `/etc/sysctl.d/99-zzz-rke2-stig-override.conf` | Override STIG base conflicts |
| NetworkManager | `/etc/NetworkManager/conf.d/rke2-canal.conf` | Ignore CNI interfaces |
| RKE2 Audit Rules | `/etc/audit/rules.d/10-rke2-server.rules` | Audit K8s components |
| fapolicyd Rules | `/etc/fapolicyd/rules.d/60-rke2.rules` | Whitelist storage paths |
| Firewalld | Service disabled and masked | RKE2 uses iptables directly |

## STIG Exemptions Required

The following STIG controls require documented exemptions when deploying RKE2
Kubernetes nodes. These exemptions must be coordinated with your Information
System Security Officer (ISSO) and documented in the System Security Plan (SSP).

### Exemption Summary

| STIG ID | Group ID | Severity | Rule Title | Justification |
|---------|----------|----------|------------|---------------|
| [RHEL-09-213105](#rhel-09-213105-user-namespaces) | V-257816 | CAT II | User namespaces must be disabled | Required for container UID isolation |
| [RHEL-09-232250](#rhel-09-232250--rhel-09-232255-orphaned-files-in-container-paths) | V-257930 | CAT II | Files must have valid group owner | Container UIDs/GIDs in /var/lib/rancher |
| [RHEL-09-232255](#rhel-09-232250--rhel-09-232255-orphaned-files-in-container-paths) | V-257931 | CAT II | Files must have valid owner | Container UIDs/GIDs in /var/lib/rancher |
| [RHEL-09-253075](#rhel-09-253075-ipv4-packet-forwarding) | V-257970 | CAT II | IPv4 packet forwarding must be disabled | Required for Kubernetes pod networking |
| [RHEL-09-251015](#rhel-09-251015--rhel-09-251020-firewalld-service) | V-257936 | CAT II | firewalld service must be active | RKE2 iptables conflicts with firewalld |
| [RHEL-09-251020](#rhel-09-251015--rhel-09-251020-firewalld-service) | V-257937 | CAT II | Firewall deny-all policy required | Consequent to RHEL-09-251015 |
| [RHEL-09-215025](#rhel-09-215025--rhel-09-215045-nfs-utils-and-gssproxy) | V-257828 | CAT II | nfs-utils must not be installed | Required for Longhorn CSI storage |
| [RHEL-09-215045](#rhel-09-215025--rhel-09-215045-nfs-utils-and-gssproxy) | V-257832 | CAT II | gssproxy must not be installed | Dependency of nfs-utils |

---

### RHEL-09-213105: User Namespaces

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-213105 |
| **Group ID** | V-257816 |
| **Severity** | CAT II (Medium) |
| **Rule Title** | RHEL 9 must disable the use of user namespaces |
| **Status** | EXEMPTION REQUIRED |

#### STIG Requirement

The STIG requires `user.max_user_namespaces = 0` to disable user namespaces,
reducing attack surface by preventing unprivileged users from creating isolated
namespace environments.

#### This Role's Setting

This role sets `user.max_user_namespaces = 15000` in `/etc/sysctl.d/98-rke2.conf`.

#### Justification

User namespaces are a **security feature** for container environments, not a
vulnerability. Disabling them would actually *reduce* container isolation and
security. Kubernetes and container runtimes use user namespaces for:

1. **Container UID Isolation**: Maps container root (UID 0) to unprivileged host
   UIDs, preventing container breakout from gaining real root privileges

2. **Rootless Container Support**: containerd and CRI-O can run containers
   without requiring host root privileges, providing defense-in-depth

3. **Kubernetes Pod User Namespaces**: Kubernetes 1.25+ includes user namespace
   support for pods (graduating from alpha to stable), enabling per-pod UID
   mapping for enhanced multi-tenant isolation

4. **Security Sandbox Enforcement**: User namespaces are foundational to
   container security models including gVisor and Kata Containers

Without user namespaces enabled, containers run with less isolation, and future
Kubernetes security features cannot be utilized.

#### Impact Assessment

| Aspect | Assessment |
|--------|------------|
| **Risk Level** | Low (enabling namespaces improves security) |
| **Attack Vector** | Namespace escapes are rare; isolation benefits outweigh risks |
| **Compensating Controls** | See below |

#### Compensating Controls

1. **SELinux Enforcing**: Mandatory access control limits namespace capabilities
2. **seccomp Profiles**: RKE2 applies default seccomp profiles restricting syscalls
3. **Pod Security Standards**: Kubernetes PSS restricts privileged containers
4. **Network Policies**: Limit pod-to-pod communication regardless of namespace
5. **Resource Quotas**: Prevent namespace resource exhaustion attacks

#### Remediation Path

| Condition | Action |
|-----------|--------|
| **If node is decommissioned** | Set `user.max_user_namespaces = 0` after removing from cluster |
| **Non-Kubernetes workloads** | Pure STIG compliance may be appropriate |
| **Kubernetes security features** | Monitor K8s pod user namespace graduation for enhanced isolation |

#### Documentation Template for ISSO

```
STIG Exemption Request

System: [System Name]
STIG ID: RHEL-09-213105
Control: Disable User Namespaces

Operational Requirement: The system is a Kubernetes (RKE2) node. User namespaces
are required for container UID isolation, which maps container root to
unprivileged host UIDs. This is a security feature that prevents container
breakout attacks from gaining real host root privileges.

Security Rationale: Disabling user namespaces would REDUCE security by:
- Allowing container root to be actual host root
- Preventing use of rootless container runtimes
- Blocking Kubernetes pod user namespace features (1.25+)

This Role's Setting: user.max_user_namespaces = 15000

Compensating Controls:
- SELinux enforcing mode provides mandatory access control
- seccomp profiles restrict available syscalls
- Kubernetes Pod Security Standards enforce container restrictions
- Network policies limit lateral movement between pods

Risk Assessment: Low. User namespaces are a defense-in-depth security control.
Enabling them improves container isolation rather than creating vulnerability.

Risk Acceptance: [ISSO Signature and Date]
```

---

### RHEL-09-232250 / RHEL-09-232255: Orphaned Files in Container Paths

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-232250 |
| **Group ID** | V-257930 |
| **Severity** | CAT II (Medium) |
| **Rule Title** | All RHEL 9 local files and directories must have a valid group owner |
| **Status** | EXEMPTION REQUIRED (for /var/lib/rancher path) |

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-232255 |
| **Group ID** | V-257931 |
| **Severity** | CAT II (Medium) |
| **Rule Title** | All RHEL 9 local files and directories must have a valid owner |
| **Status** | EXEMPTION REQUIRED (for /var/lib/rancher path) |

#### STIG Requirement

These STIGs require all files on the system to have valid owner and group owner
entries that exist in `/etc/passwd` and `/etc/group` respectively. Automated
scanners flag files without matching local user/group entries.

#### This Role's Behavior

This role **excludes** the following paths from orphaned file checks:

1. **Container Data** (`rhel_rke2_stig_rancher_base_path`): Always excluded
   (default: `/var/lib/rancher`)

2. **Domain User Homes** (`rhel_rke2_stig_domain_home_path`): Excluded when
   `rhel_rke2_stig_sssd_enabled: true` (example: `/home/contoso.com`)

When the optional remediation is enabled (`rhel_rke2_stig_orphan_files_enabled:
true`), the role will find and fix orphaned files on the rest of the system but
intentionally skip paths where non-local UIDs/GIDs are expected.

#### Justification

Files under the excluded paths contain UIDs and GIDs that **intentionally do
not exist in local `/etc/passwd` and `/etc/group`**. This is a security feature,
not a misconfiguration:

1. **User Namespace Isolation**: Containers run with mapped UIDs (e.g., UID
   100000-165535) that provide isolation from host users. These UIDs appear as
   "orphaned" because they don't exist in `/etc/passwd` - by design.

2. **Container Image UIDs**: Container images define their own users (e.g.,
   `nginx` user, `postgres` user) with UIDs that differ from host system users.
   These are baked into the container image and managed by the container runtime.

3. **Kubernetes Data Directories**: The following paths contain container data:
   - `/var/lib/rancher/rke2` - RKE2 Kubernetes data and containerd layers
   - `/var/lib/rancher/k3s` - K3S Kubernetes data and containerd layers
   - `/var/lib/rancher/longhorn` - Longhorn CSI persistent volume data
   - `/var/lib/rancher/local-path-provisioner` - Local-path PV data

4. **Domain User Home Directories**: When systems are joined to Active Directory
   or FreeIPA, user home directories (e.g., `/home/contoso.com/jsmith`) are
   owned by UIDs from the directory service. These UIDs are resolved via SSSD
   at runtime but don't exist in local `/etc/passwd`.

5. **Remediating Would Break Systems**: Changing ownership to `root:root` would:
   - Break running containers and corrupt persistent volumes
   - Lock domain users out of their home directories
   - Cause application failures for any process expecting specific ownership

#### Automated Scanner Considerations

**Nessus, SCAP, and OpenSCAP scanners** will flag files under `/var/lib/rancher`
and domain user home directories as findings for RHEL-09-232250 and
RHEL-09-232255. These are **false positives** for Kubernetes environments with
directory service integration. Provide assessors with this documentation.

#### Verification Commands for Assessors

```bash
# Show that files under /var/lib/rancher have container UIDs
find /var/lib/rancher -maxdepth 3 -nouser -o -nogroup 2>/dev/null | head -20

# Show domain user home directory ownership (UIDs from AD/IPA)
ls -ln /home/contoso.com/ 2>/dev/null | head -10

# Show the UID range used by user namespaces (if enabled)
cat /proc/sys/kernel/overflowuid

# Verify SSSD is resolving directory users
getent passwd $(ls /home/contoso.com/ 2>/dev/null | head -1) 2>/dev/null

# Verify the rest of the system has no orphaned files (excluding known paths)
find / -xdev -path /var/lib/rancher -prune -o -path /home/contoso.com -prune \
  -o \( -nouser -o -nogroup \) -print 2>/dev/null
```

#### Impact Assessment

| Aspect | Assessment |
|--------|------------|
| **Risk Level** | Low |
| **Attack Vector** | None - container UIDs provide security isolation |
| **Compensating Controls** | See below |

#### Compensating Controls

1. **User Namespaces**: Container UIDs are mapped to unprivileged host UID
   ranges, preventing container breakout from gaining host privileges
2. **SELinux Enforcement**: Mandatory access control restricts container access
   regardless of UID
3. **Read-Only Root Filesystem**: Many containers run with read-only root
   filesystems, limiting what can be modified
4. **Kubernetes RBAC**: Access to persistent volumes is controlled by Kubernetes
   PersistentVolumeClaim bindings
5. **seccomp Profiles**: Restrict syscalls available to containers
6. **SSSD UID Resolution**: Domain user UIDs are resolved at runtime via SSSD,
   providing centralized identity management and audit trails

#### Remediation Path

| Condition | Action |
|-----------|--------|
| **If node is decommissioned** | Run `find / -nouser -o -nogroup` after removing from cluster and leaving domain |
| **Container path differs** | Set `rhel_rke2_stig_rancher_base_path` to match your deployment |
| **Domain home path differs** | Set `rhel_rke2_stig_domain_home_path` to match your AD/IPA realm |

#### Configuration

Override the exclusion paths in your playbook or inventory:

```yaml
# Container data exclusion path (always excluded)
rhel_rke2_stig_rancher_base_path: /var/lib/rancher

# Domain user home directory exclusion (excluded when SSSD enabled)
rhel_rke2_stig_sssd_enabled: true
rhel_rke2_stig_domain_home_path: "/home/contoso.com"

# Enable orphan file remediation (excludes above paths)
rhel_rke2_stig_orphan_files_enabled: true
```

#### Documentation Template for ISSO

```
STIG Exemption Request

System: [System Name]
STIG IDs: RHEL-09-232250, RHEL-09-232255
Controls: Valid File Owner, Valid Group Owner

Operational Requirement: The system is a Kubernetes (RKE2/K3S) node joined to
Active Directory/FreeIPA. Two categories of files have UIDs/GIDs that do not
exist in local /etc/passwd and /etc/group:

1. Container Data (/var/lib/rancher): Container filesystems use mapped UIDs
   from user namespace isolation. This is a security feature that prevents
   container breakout by mapping container root to unprivileged host UIDs.

2. Domain User Homes (/home/<DOMAIN>): User home directories are owned by
   UIDs from Active Directory/FreeIPA. These UIDs are resolved at runtime
   via SSSD but don't exist in local passwd files.

Scope of Exemption:
- /var/lib/rancher directory tree (container data)
- /home/<DOMAIN> directory tree (domain user homes, if SSSD enabled)
All other paths are checked and remediated per STIG requirements.

Security Rationale:
- Container UIDs that don't exist on the host provide BETTER security through
  isolation. Remediating would break container isolation and corrupt workloads.
- Domain user UIDs are managed centrally in AD/IPA with proper access controls.
  Remediating would lock users out of their home directories.

Compensating Controls:
- User namespaces map container UIDs to unprivileged host ranges
- SELinux enforcing mode provides mandatory access control
- Kubernetes RBAC controls access to persistent volumes
- seccomp profiles restrict container syscalls
- SSSD provides centralized identity management with audit logging
- AD/IPA group policies enforce access controls on domain users

Risk Assessment: Low. Non-local UIDs in excluded paths are security features:
- Container UIDs provide isolation (not a vulnerability)
- Domain UIDs are managed centrally with proper access controls
All other system paths are checked and remediated per STIG requirements.

Risk Acceptance: [ISSO Signature and Date]
```

---

### RHEL-09-253075: IPv4 Packet Forwarding

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-253075 |
| **Group ID** | V-257970 |
| **Severity** | CAT II (Medium) |
| **Rule Title** | RHEL 9 must not enable IPv4 packet forwarding unless the system is a router |
| **Status** | EXEMPTION REQUIRED |

#### STIG Requirement

The STIG requires `net.ipv4.ip_forward = 0` and `net.ipv4.conf.all.forwarding = 0`
to prevent the system from forwarding IPv4 packets unless it is designated as a
router.

#### This Role's Setting

This role configures IP forwarding in **two locations** to ensure settings persist
across all scenarios:

**1. Direct `/etc/sysctl.conf` Modifications (PRIMARY - using `ansible.posix.sysctl`):**

```ini
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.default.forwarding = 1
```

**2. Defense-in-Depth Configuration Files:**

- `/etc/sysctl.d/98-rke2.conf` - RKE2 tuning and initial forwarding settings
- `/etc/sysctl.d/99-zzz-rke2-stig-override.conf` - Documentation and backup settings

**Critical Settings:**

| Setting | Purpose |
|---------|---------|
| `net.ipv4.ip_forward = 1` | Global master switch - MUST be enabled for any forwarding |
| `net.ipv4.conf.all.forwarding = 1` | Per-interface control for all existing interfaces |
| `net.ipv4.conf.default.forwarding = 1` | Per-interface control for new interfaces |

> **Note:** Both the global switch (`ip_forward`) and per-interface settings
> (`conf.*.forwarding`) must be enabled. If the global switch is `0`, per-interface
> settings have no effect.

#### Critical: sysctl Load Order

**IMPORTANT:** The `sysctl --system` command processes configuration in this order:

1. `/run/sysctl.d/*.conf`
2. `/etc/sysctl.d/*.conf` (alphabetically)
3. `/usr/local/lib/sysctl.d/*.conf`
4. `/usr/lib/sysctl.d/*.conf`
5. **`/etc/sysctl.conf` (ALWAYS LAST)**

This means `/etc/sysctl.conf` **always overrides** settings from `/etc/sysctl.d/*.conf`,
regardless of file naming (even `99-zzz-*` prefixes).

**Why this matters:** When the STIG base remediation playbook runs, it sets
`net.ipv4.conf.all.forwarding = 0` in `/etc/sysctl.conf`. If we only deploy
settings to `/etc/sysctl.d/`, the STIG base settings in `/etc/sysctl.conf` will
override them on every `sysctl --system` reload.

**This role's solution:** Use `ansible.posix.sysctl` to write our exempted settings
directly to `/etc/sysctl.conf`. This ensures our Kubernetes-required settings are
in the file that gets processed **last**, taking final precedence.

The `/etc/sysctl.d/99-zzz-rke2-stig-override.conf` file is retained for:

- Documentation and audit trails
- Defense-in-depth (if `/etc/sysctl.conf` is regenerated)
- Clear visibility of what settings are being overridden and why

#### Justification

Kubernetes nodes **require** IPv4 packet forwarding to be enabled for pod-to-pod
networking to function. The node acts as a router for container network traffic
in the CNI (Container Network Interface) overlay network. Without IP forwarding:

- Pods cannot communicate across nodes
- Services cannot route traffic to backend pods
- The Kubernetes cluster is non-functional

This is a fundamental architectural requirement of Kubernetes networking, not a
misconfiguration.

#### Impact Assessment

| Aspect | Assessment |
|--------|------------|
| **Risk Level** | Low |
| **Attack Vector** | Local network only; requires existing network access |
| **Compensating Controls** | See below |

#### Compensating Controls

The following controls mitigate the risk of enabling IP forwarding:

1. **Network Segmentation**: RKE2 nodes are deployed on isolated network segments
   with firewall rules restricting traffic to authorized sources only
2. **CNI Network Policies**: Kubernetes NetworkPolicy resources restrict pod-to-pod
   communication to authorized flows
3. **Source Routing Disabled**: The following STIG-compliant settings prevent
   source-routed packet exploitation:
   - `net.ipv4.conf.all.accept_source_route = 0`
   - `net.ipv4.conf.default.accept_source_route = 0`
4. **ICMP Redirects Disabled**: Redirect acceptance is disabled per STIG requirements
5. **RKE2 iptables Rules**: RKE2 implements iptables-based network filtering that
   restricts traffic to required Kubernetes ports (see RHEL-09-251015 exemption)

#### Remediation Path

| Condition | Action |
|-----------|--------|
| **If node is decommissioned** | Remove from Kubernetes cluster and disable IP forwarding |
| **If STIG is updated** | Review for Kubernetes-specific guidance that may supersede this exemption |
| **Alternative** | None available - IP forwarding is architecturally required for Kubernetes |

#### STIG Reference

Per the STIG check content: *"If the IPv4 forwarding value is not '0' **and is
not documented with the information system security officer (ISSO) as an
operational requirement**, this is a finding."*

This exemption documents the operational requirement as permitted by the STIG.

#### Documentation Template for ISSO

```
STIG Exemption Request

System: [System Name]
STIG ID: RHEL-09-253075
Control: IPv4 Packet Forwarding

Operational Requirement: The system is a Kubernetes (RKE2) node that requires
IPv4 packet forwarding for container networking. Kubernetes CNI networking
fundamentally depends on the node forwarding packets between the pod network
overlay and external networks.

Compensating Controls:
- Network segmentation isolates RKE2 nodes
- Kubernetes NetworkPolicy restricts pod communication
- Source routing and ICMP redirects remain disabled per STIG
- RKE2 iptables rules provide network filtering (see RHEL-09-251015 exemption)

Risk Acceptance: [ISSO Signature and Date]
```

### RHEL-09-251015 / RHEL-09-251020: Firewalld Service

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-251015 |
| **Group ID** | V-257936 |
| **Severity** | CAT II (Medium) |
| **Rule Title** | The firewalld service on RHEL 9 must be active |
| **Status** | EXEMPTION REQUIRED |

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-251020 |
| **Group ID** | V-257937 |
| **Severity** | CAT II (Medium) |
| **Rule Title** | The RHEL 9 firewall must employ a deny-all, allow-by-exception policy |
| **Status** | EXEMPTION REQUIRED (consequent to RHEL-09-251015) |

#### STIG Requirement

RHEL-09-251015 requires the firewalld service to be active and running.
RHEL-09-251020 requires firewalld to be configured with a deny-all, allow-by-
exception policy using firewall zones.

#### This Role's Setting

This role stops, disables, and masks the firewalld service.

#### Justification

RKE2 Kubernetes uses **iptables directly** to manage pod networking, service
load balancing, and network policy enforcement. Running firewalld concurrently
with RKE2's iptables management causes:

- **Rule conflicts**: firewalld and RKE2 both manipulate iptables, leading to
  unpredictable rule ordering and dropped packets
- **Silent failures**: Cluster networking fails intermittently in ways that are
  difficult to diagnose (pods cannot communicate, services become unreachable)
- **Race conditions**: Both systems attempt to manage the same iptables chains,
  causing rules to be overwritten or deleted unexpectedly

Despite extensive testing with firewalld configurations (including custom zones,
direct rules, and rich rules), stable coexistence has not been achievable. This
is a known limitation documented by Rancher and other Kubernetes distributions.

**RKE2's iptables rules ARE a firewall.** They implement:
- Default deny for pod ingress (unless allowed by NetworkPolicy)
- SNAT/DNAT for service load balancing
- Port filtering based on defined Kubernetes services
- Network isolation between namespaces (when NetworkPolicy is configured)

#### Impact Assessment

| Aspect | Assessment |
|--------|------------|
| **Risk Level** | Low |
| **Attack Vector** | Network-based; mitigated by external controls |
| **Compensating Controls** | See below |

#### Compensating Controls

1. **RKE2 iptables Firewall**: RKE2 implements comprehensive iptables rules that
   filter traffic to only allowed Kubernetes services and pods
2. **Kubernetes NetworkPolicy**: CNI-level network policies restrict pod-to-pod
   communication based on labels, namespaces, and ports
3. **Network Segmentation**: RKE2 nodes are deployed on isolated network segments
   with upstream firewall/ACLs restricting access to management and service ports
4. **Ingress Controllers**: External traffic enters only through designated ingress
   points with TLS termination and request filtering
5. **Service Mesh** (if deployed): mTLS between services provides encryption and
   authentication at the application layer
6. **API Server Authentication**: Kubernetes API requires valid certificates and
   RBAC authorization for all requests

#### Ports Exposed by RKE2 (Filtered by iptables)

| Port | Protocol | Purpose | Exposure |
|------|----------|---------|----------|
| 6443 | TCP | Kubernetes API | Control plane |
| 9345 | TCP | RKE2 supervisor | Control plane |
| 10250 | TCP | Kubelet API | All nodes |
| 2379-2380 | TCP | etcd | Control plane |
| 8472 | UDP | VXLAN (Canal) | All nodes |
| 30000-32767 | TCP/UDP | NodePort services | As configured |

#### Remediation Path

| Condition | Action |
|-----------|--------|
| **If node is decommissioned** | Re-enable firewalld after removing from cluster |
| **If RKE2 adds firewalld support** | Review Rancher documentation for integration guidance |
| **Alternative** | External network firewall/ACLs provide perimeter filtering |

#### Documentation Template for ISSO

```
STIG Exemption Request

System: [System Name]
STIG IDs: RHEL-09-251015, RHEL-09-251020
Controls: Firewalld Service Active, Deny-All Firewall Policy

Operational Requirement: The system is a Kubernetes (RKE2) node. RKE2 manages
network filtering directly via iptables for pod networking, service routing,
and network policy enforcement. Running firewalld concurrently causes iptables
rule conflicts resulting in cluster networking failures.

Compensating Controls:
- RKE2 iptables rules provide network filtering for all K8s traffic
- Kubernetes NetworkPolicy enforces pod-to-pod communication restrictions
- Network segmentation with upstream firewalls restricts node access
- Only required Kubernetes ports are exposed (6443, 9345, 10250, etc.)

Technical Justification: RKE2 and firewalld both manage iptables chains. Their
concurrent operation causes rule conflicts, race conditions, and silent
networking failures. This incompatibility is documented by Rancher (RKE2
vendor) and is a known limitation of running firewalld with Kubernetes.

Risk Acceptance: [ISSO Signature and Date]
```

### RHEL-09-215025 / RHEL-09-215045: nfs-utils and gssproxy

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-215025 |
| **Group ID** | V-257828 |
| **Severity** | CAT II (Medium) |
| **Rule Title** | RHEL 9 must not have the nfs-utils package installed |
| **Status** | EXEMPTION REQUIRED |

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-215045 |
| **Group ID** | V-257832 |
| **Severity** | CAT II (Medium) |
| **Rule Title** | RHEL 9 must not have the gssproxy package installed |
| **Status** | EXEMPTION REQUIRED (dependency of nfs-utils) |

#### STIG Requirement

RHEL-09-215025 prohibits installation of the `nfs-utils` package to reduce the
attack surface by removing Network File System client functionality.

RHEL-09-215045 prohibits installation of the `gssproxy` package, which provides
GSS-API credential handling. This package is installed as a dependency of
nfs-utils.

#### This Role's Requirement

RKE2 nodes require `nfs-utils` to be installed for Longhorn CSI persistent
storage functionality.

#### Justification

**Longhorn** is the CNCF-graduated storage solution used for persistent volumes
in DoD Kubernetes deployments. Longhorn requires NFS utilities for:

1. **ReadWriteMany (RWX) Volumes**: Longhorn uses NFS to export volumes that
   need to be mounted by multiple pods simultaneously (RWX access mode). This is
   essential for:
   - Shared application data
   - Centralized logging
   - Distributed caching
   - Stateful applications requiring shared storage

2. **Backup and Restore**: Longhorn's backup targets (S3-compatible, NFS) use
   nfs-utils for NFS-based backup destinations in airgapped environments where
   S3 may not be available.

3. **Data Locality**: NFS utilities enable Longhorn to efficiently manage
   replica data across nodes.

Without `nfs-utils`, Longhorn cannot:
- Provide RWX persistent volumes
- Export shared storage to pods
- Complete NFS-based backup operations

This would severely limit the storage capabilities of the Kubernetes cluster,
preventing deployment of many stateful workloads that require shared storage.

**gssproxy** is automatically installed as a dependency of nfs-utils. While not
directly used by Longhorn (which does not require Kerberos authentication for
its internal NFS shares), it cannot be removed without breaking the nfs-utils
package installation.

#### Impact Assessment

| Aspect | Assessment |
|--------|------------|
| **Risk Level** | Low |
| **Attack Vector** | Network-based; requires access to NFS ports |
| **Compensating Controls** | See below |

#### Compensating Controls

1. **No External NFS Exports**: The NFS services exposed by Longhorn are
   internal to the Kubernetes cluster and are not accessible from external
   networks. Longhorn manages NFS shares for pod-to-pod storage only.

2. **Network Segmentation**: RKE2 nodes are deployed on isolated network
   segments. NFS traffic (ports 2049, 111) is restricted to cluster-internal
   communication via Kubernetes NetworkPolicy and upstream firewall rules.

3. **Longhorn Encryption**: Longhorn supports volume encryption at rest using
   Linux kernel dm-crypt, protecting data even if NFS traffic were intercepted.

4. **No rpcbind Exposure**: The `rpcbind` service is not exposed externally.
   Longhorn's NFS implementation uses fixed ports that do not require portmapper
   services.

5. **SELinux Enforcement**: SELinux remains in enforcing mode, providing
   mandatory access control for NFS-related operations.

6. **Kubernetes RBAC**: Access to Longhorn volumes is controlled via Kubernetes
   RBAC and PersistentVolumeClaim (PVC) bindings. Only authorized pods can mount
   Longhorn NFS shares.

7. **gssproxy Inactive**: While installed, the gssproxy service is not enabled
   or running unless explicitly configured for Kerberos authentication.

#### NFS Ports (Controlled by Kubernetes Network Policy)

| Port | Protocol | Purpose | Exposure |
|------|----------|---------|----------|
| 2049 | TCP/UDP | NFSv4 | Cluster-internal only |
| 111 | TCP/UDP | rpcbind (if used) | Cluster-internal only |
| 20048 | TCP/UDP | mountd | Cluster-internal only |

#### Remediation Path

| Condition | Action |
|-----------|--------|
| **If Longhorn not required** | Remove nfs-utils and gssproxy packages |
| **If alternative CSI available** | Evaluate CSI drivers that don't require NFS (e.g., local-path for RWO only) |
| **If STIG updated** | Review for Kubernetes storage-specific guidance |

#### Alternative Considered

**Local Path Provisioner**: Provides only ReadWriteOnce (RWO) volumes. Does not
require nfs-utils but cannot support workloads requiring shared storage (RWX).
This is insufficient for many production workloads.

#### Documentation Template for ISSO

```
STIG Exemption Request

System: [System Name]
STIG IDs: RHEL-09-215025, RHEL-09-215045
Controls: nfs-utils Package, gssproxy Package

Operational Requirement: The system is a Kubernetes (RKE2) node using Longhorn
CSI for persistent storage. Longhorn requires nfs-utils to provide ReadWriteMany
(RWX) persistent volumes, which are essential for stateful applications requiring
shared storage across multiple pods.

Package Justification:
- nfs-utils: Required by Longhorn CSI for RWX volume support and NFS-based
  backup targets in airgapped environments.
- gssproxy: Installed as a dependency of nfs-utils. Not actively used but
  cannot be removed without breaking nfs-utils functionality.

Compensating Controls:
- NFS shares are internal to Kubernetes cluster only (not externally exposed)
- Network segmentation restricts NFS ports to cluster-internal traffic
- Longhorn volume encryption protects data at rest
- SELinux enforcing mode provides mandatory access control
- Kubernetes RBAC controls access to persistent volumes
- gssproxy service remains disabled

Risk Assessment: Low. NFS functionality is containerized within Longhorn and
not exposed to external networks. Attack surface is limited to cluster-internal
pod-to-pod communication.

Risk Acceptance: [ISSO Signature and Date]
```

### RHEL-09-215105: FIPS Cryptographic Policy

| Field | Value |
|-------|-------|
| **STIG ID** | RHEL-09-215105 |
| **Group ID** | V-258241 |
| **Severity** | CAT I (High) - raised from CAT II in RHEL 9 STIG V2R8 (01 Apr 2026); check/fix text unchanged |
| **Rule Title** | RHEL 9 must implement a FIPS 140-3-compliant systemwide cryptographic policy |
| **Status** | COMPLIANT (with documentation for assessors) |

> **CAT I note:** As of RHEL 9 STIG V2R8 this is a CAT I (High) finding (DISA
> elevated the risk rating; the technical requirement did not change). The
> `FIPS:AD-SUPPORT` rationale below documents this as a **compliant
> configuration, not an exemption** - the STIG check explicitly accepts any
> `FIPS:<subpolicy>` form. Active Directory interoperability is mandatory in
> the target environment, and the RC4-for-Kerberos behavior the subpolicy
> introduces is documented here for assessor transparency.

#### STIG Requirement

The STIG requires RHEL 9 to use a FIPS 140-3 compliant cryptographic policy,
typically achieved by running `update-crypto-policies --set FIPS`.

#### This Role's Setting

This role sets `FIPS:AD-SUPPORT` by default via the variable
`rhel_rke2_stig_crypto_policy`. This can be changed in `defaults/main.yml` or
overridden in playbooks.

```yaml
# Default setting (recommended for AD/IPA environments)
rhel_rke2_stig_crypto_policy: "FIPS:AD-SUPPORT"

# Pure FIPS (only if no AD/IPA integration required)
rhel_rke2_stig_crypto_policy: "FIPS"
```

#### Why This Is Still FIPS Compliant

**This is NOT an exemption in the traditional sense.** The `FIPS:AD-SUPPORT`
policy maintains full FIPS 140-3 compliance at the cryptographic module level.
Understanding this requires distinguishing between two concepts:

| Concept | Description |
|---------|-------------|
| **FIPS Mode** | Kernel and OpenSSL cryptographic modules operate in FIPS-validated mode |
| **Crypto Policy** | Application-level configuration for which algorithms to *prefer* |

When `FIPS:AD-SUPPORT` is set:

1. **The kernel crypto module remains FIPS-validated** (`/proc/sys/crypto/fips_enabled = 1`)
2. **OpenSSL operates in FIPS mode** (only validated algorithms for TLS, SSH, etc.)
3. **Only Kerberos (krb5) gets the RC4 exception** for Active Directory compatibility
4. **All other protocols remain FIPS-strict** (TLS 1.2+, AES-GCM, SHA-256+)

#### The AD Compatibility Problem

Active Directory Kerberos authentication presents a compatibility challenge:

| AD Version | Default Kerberos Encryption | FIPS Status |
|------------|----------------------------|-------------|
| Server 2008/2012 | RC4-HMAC (arcfour) | Not FIPS-approved |
| Server 2016 | AES + RC4 fallback | AES is compliant |
| Server 2019+ | AES preferred | Compliant |

Even modern AD environments may use RC4 for:
- Cross-realm trust tickets (IPA ↔ AD)
- Backward compatibility with older domain members
- Specific service account configurations

Without RC4 for Kerberos, systems cannot:
- Join AD domains via `realm join`
- Authenticate users via SSSD against AD
- Establish IPA-AD trust relationships

#### What FIPS:AD-SUPPORT Actually Enables

The `AD-SUPPORT` subpolicy modifies *only* Kerberos settings:

```
# From /usr/share/crypto-policies/policies/modules/AD-SUPPORT.pmod
# Enables RC4 for Kerberos ONLY - not for TLS, SSH, or other protocols

cipher@kerberos = AES-256-GCM+ AES-256-CCM AES-256-CBC AES-128-GCM+ \
                  AES-128-CCM AES-128-CBC RC4
```

| Protocol | RC4 Allowed? | SHA-1 Allowed? |
|----------|--------------|----------------|
| TLS/HTTPS | No | No |
| SSH | No | No |
| Kerberos | Yes (for AD) | Limited |
| IPsec | No | No |

#### Automated Scanner Considerations

**Nessus, SCAP, and similar automated scanners** may flag `FIPS:AD-SUPPORT` as
non-compliant because they perform simple string matching:

```bash
# Scanner checks for this exact value:
update-crypto-policies --show
# Expected: FIPS
# Actual: FIPS:AD-SUPPORT
# Result: FAIL (false positive)
```

**This is a scanner limitation, not a compliance failure.** Provide assessors
with this documentation to explain:

1. The cryptographic modules are FIPS-validated (check `/proc/sys/crypto/fips_enabled`)
2. The subpolicy only affects Kerberos, not TLS/SSH
3. This is Red Hat's official solution for AD-integrated FIPS systems
4. The alternative (pure FIPS) breaks AD authentication entirely

#### Verification Commands for Assessors

```bash
# Verify FIPS mode is enabled at kernel level
cat /proc/sys/crypto/fips_enabled
# Expected output: 1

# Verify crypto policy
update-crypto-policies --show
# Expected output: FIPS:AD-SUPPORT

# Verify OpenSSL is in FIPS mode
openssl list -providers | grep -i fips
# Expected: fips provider should be listed

# Show what the AD-SUPPORT subpolicy modifies
cat /usr/share/crypto-policies/policies/modules/AD-SUPPORT.pmod

# Verify Kerberos can use AES (primary) and RC4 (fallback)
grep -E "permitted_enctypes|default_tkt_enctypes" /etc/krb5.conf
```

#### Impact Assessment

| Aspect | Assessment |
|--------|------------|
| **Risk Level** | Low |
| **FIPS Compliance** | Maintained (cryptographic modules are FIPS-validated) |
| **Attack Vector** | RC4 weakness limited to Kerberos tickets only |
| **Compensating Controls** | See below |

#### Compensating Controls

1. **AES Preferred**: Modern AD (2016+) negotiates AES-256 by default; RC4 is
   fallback only
2. **Kerberos Ticket Lifetime**: Tickets expire (typically 10 hours), limiting
   exposure window for any RC4-encrypted tickets
3. **Network Segmentation**: Kerberos traffic restricted to management networks
4. **Monitoring**: Authentication logs capture ticket encryption types for audit
5. **AD Hardening**: Disable RC4 on AD side once all clients support AES-only

#### Remediation Path

| Condition | Action |
|-----------|--------|
| **AD upgraded to 2019+** | Test pure `FIPS` policy; disable if auth works |
| **No AD integration** | Set `rhel_rke2_stig_crypto_policy: "FIPS"` |
| **IPA-only (no AD trust)** | Pure `FIPS` may work; test before deploying |

#### Configuration

Override the default in your playbook or inventory:

```yaml
# For pure FIPS (no AD)
rhel_rke2_stig_crypto_policy: "FIPS"

# For FIPS with AD (default)
rhel_rke2_stig_crypto_policy: "FIPS:AD-SUPPORT"

# To disable crypto policy management entirely
rhel_rke2_stig_crypto_policy_enabled: false
```

#### Documentation Template for Assessors

```
STIG Finding Clarification

System: [System Name]
STIG ID: RHEL-09-215105
Scanner Finding: FAIL - Crypto policy is FIPS:AD-SUPPORT, not FIPS

Clarification: This is a FALSE POSITIVE. The system IS FIPS 140-3 compliant.

Technical Explanation:
1. FIPS mode is enabled: /proc/sys/crypto/fips_enabled = 1
2. OpenSSL operates in FIPS-validated mode
3. The AD-SUPPORT subpolicy ONLY modifies Kerberos (krb5) settings
4. All other protocols (TLS, SSH, IPsec) remain FIPS-strict

Operational Requirement:
The system integrates with Active Directory via [IPA trust / direct join / SSSD].
Active Directory Kerberos requires RC4 (arcfour-hmac) for authentication in
environments with legacy domain controllers or cross-realm trusts.

Verification:
- Kernel FIPS: cat /proc/sys/crypto/fips_enabled → 1
- OpenSSL FIPS: openssl list -providers → shows fips
- TLS still FIPS-only: openssl ciphers -v | grep -v AES → no RC4

This configuration follows Red Hat's official guidance for FIPS-compliant
systems requiring Active Directory integration.

Reference: Red Hat KB article "Using AD-SUPPORT crypto policy subpolicy"
https://access.redhat.com/solutions/6985023

Assessor Acknowledgment: [Signature and Date]
```

## Dependencies

None. This role is designed to be standalone.

## Example Playbook

### Basic Usage

```yaml
---
- name: Apply STIG compliance to RKE2 nodes
  hosts: rke2_nodes
  become: true
  roles:
    - role: darkhonor.rhel_rke2_stig
```

### With Tags

```yaml
---
- name: Apply specific STIG controls
  hosts: rke2_nodes
  become: true
  roles:
    - role: darkhonor.rhel_rke2_stig
      tags:
        - audit
        - kernel
```

### Running Specific Tags

```bash
# Apply only audit rules
ansible-playbook site.yml --tags audit

# Apply only kernel module blacklist
ansible-playbook site.yml --tags kernel

# Apply specific STIG finding
ansible-playbook site.yml --tags RHEL-09-611075
```

## Files Deployed

| File | Description |
|------|-------------|
| `/etc/audit/rules.d/20-stig.rules` | STIG-compliant audit rules |
| `/etc/audit/rules.d/10-rke2-server.rules` | RKE2-specific audit rules |
| `/etc/modprobe.d/blacklist.conf` | Kernel module blacklist |
| `/etc/modules-load.d/rke2-modules.conf` | Kernel modules for K8s networking |
| `/etc/sysctl.d/98-rke2.conf` | Kernel tuning for Kubernetes |
| `/etc/sysctl.d/99-zzz-rke2-stig-override.conf` | Override STIG base sysctl conflicts |
| `/etc/NetworkManager/conf.d/rke2-canal.conf` | CNI interface ignore list |
| `/etc/fapolicyd/rules.d/60-rke2.rules` | Application whitelist for storage |

## Development Setup

### Pre-commit Hooks

This repository uses pre-commit hooks to catch issues before pushing. Every
hook runs a tool from the hash-pinned `requirements.txt`, so install that
first (see [CONTRIBUTING.md](CONTRIBUTING.md) for the full contract):

```bash
# Install the hash-pinned toolchain (pre-commit and every hook's tool)
python -m pip install --require-hashes -r requirements.txt

# Install the Ansible collections the role and its lint run need
ansible-galaxy collection install -r requirements.yml

# Install the git hooks with that pre-commit
python -m pre_commit install

# Run against all files (what the CI lint job runs)
pre-commit run --all-files
```

Once installed, yamllint, ansible-lint, and the file-hygiene hooks run on
every commit.

## Testing

This role is tested using Molecule with Podman:

```bash
# Install test dependencies (hash-pinned lock)
python -m pip install --require-hashes -r requirements.txt

# Run full test suite
molecule test

# Run converge only (apply role)
molecule converge

# Run verification tests
molecule verify

# Clean up test environment
molecule destroy
```

### CI/CD

Automated testing runs on every push and pull request:

- **Linting:** `pre-commit run --all-files` (yamllint, ansible-lint, file
  hygiene hooks) and actionlint
- **Syntax:** ansible-playbook --syntax-check
- **Integration:** Molecule with Rocky Linux 9 container (the RHEL 10 path is
  live-validated on Rocky 10; a Rocky 10 Molecule platform is a planned follow-up)
- **Security:** Trivy vulnerability scanning

## Compliance References

- [RHEL 9 STIG V2R8](https://public.cyber.mil/stigs/) - OS hardening, RHEL 9 targets (Released: 01 Apr 2026)
- [RHEL 10 STIG V1R1](https://public.cyber.mil/stigs/) - OS hardening, RHEL 10 targets
- [RKE2 STIG V2R6](https://public.cyber.mil/stigs/) - Kubernetes controls (Released: 01 Apr 2026)
- [Container Platform SRG V2R4](https://public.cyber.mil/stigs/) - Container runtime (Released: 28 Oct 2025)
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) - Control framework
- [FIPS 140-2/140-3](https://csrc.nist.gov/projects/cryptographic-module-validation-program) - Cryptographic requirements
- [Automated Scanning Analysis](AutomatedScans.md) - OpenSCAP gaps, false positives, and manual checks

## RKE2/K3S Considerations

When deploying this role to Kubernetes nodes, be aware of:

- **Firewall:** RKE2 requires specific ports (6443, 9345, 10250, 2379-2380)
- **SELinux:** Must remain in enforcing mode; RKE2 policies may need tuning
- **Audit volume:** Kubernetes syscalls generate high audit log volume
- **etcd:** Requires separate TLS and backup configuration

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for
guidelines.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/new-stig-control`)
3. Commit your changes (`git commit -am 'Add RHEL-09-XXXXXX control'`)
4. Push to the branch (`git push origin feature/new-stig-control`)
5. Open a Pull Request

## STIG Release History

This section tracks changes between DISA STIG releases to help identify scanner
findings from outdated baselines and understand rule evolution. See
`stigs/TODO.md` for items affected by recent changes.

> **Automated Scanning:** For detailed analysis of OpenSCAP scanner gaps,
> false positives, and manual check scripts for rules not covered by automated
> scanners, see [AutomatedScans.md](AutomatedScans.md).

### RHEL 9 STIG: V2R7 → V2R8 (April 2026)

| STIG ID | Change | Category | Description |
|---------|--------|----------|-------------|
| RHEL-09-215105 | Severity | CAT II → CAT I | FIPS systemwide crypto policy (implemented) |
| RHEL-09-255070 | Severity | CAT II → CAT I | SSH client FIPS MACs (implemented) |
| RHEL-09-255075 | Severity | CAT II → CAT I | SSH server FIPS MACs (implemented) |

**26 rules** modified (0 added, 0 removed). 8 FIPS/crypto rules were raised from
CAT II to CAT I; 3 of those are implemented by this role (above). Check/fix text
is unchanged, so remediation logic is unaffected. Only the risk rating and the
role's severity metadata were updated.

### RKE2 STIG: V2R5 → V2R6 (April 2026)

5 rules received check/fix procedure updates (0 added, 0 removed, no severity
changes). None are implemented by this role; no impact to coverage.

### RHEL 9 STIG: V2R6 → V2R7 (January 2026)

| STIG ID | Change | Category | Description |
|---------|--------|----------|-------------|
| RHEL-09-654097 | Added | CAT II | Audit cron/at scripts and executables |
| RHEL-09-411115 | Removed | CAT II | Init files world-writable check (deprecated) |
| RHEL-09-412075 | Removed | CAT III | Last login date/time display (deprecated) |

**53 rules** received check/fix procedure updates (no severity or title changes).
Key areas updated: kernel module checks, sysctl verification, filesystem permissions.

### RHEL 9 STIG: V2R5 → V2R6 (October 2025)

| STIG ID | Change | Category | Description |
|---------|--------|----------|-------------|
| RHEL-09-654096 | Removed | CAT II | Audit cron scripts (replaced by 654097 in V2R7) |
| RHEL-09-654260 | Removed | CAT II | Audit account creation/modification |
| RHEL-09-215035 | Modified | CAT II | Title updated (telnet-server package) |
| RHEL-09-653040 | Modified | CAT II | Title updated (audit log permissions) |

Net change: 449 → 447 rules (2 removed, 0 added).

### RKE2 STIG: V2R4 → V2R5 (January 2026)

| STIG ID | Change | Category | Description |
|---------|--------|----------|-------------|
| CNTR-R2-000120 | Removed | CAT I | Insecure port disabled (deprecated) |
| CNTR-R2-000140 | Removed | CAT I | Insecure bind address (deprecated) |
| CNTR-R2-001500 | Removed | CAT I | Keystore encryption (deprecated) |
| CNTR-R2-000520 | Modified | CAT II | Check procedure updated |
| CNTR-R2-001130 | Modified | CAT II | Check/Fix procedures updated |

Net change: 24 → 21 rules (3 CAT I removed). Significant reduction in critical findings.

### Container Platform SRG: V2R3 → V2R4 (October 2025)

| STIG ID | Change | Category | Description |
|---------|--------|----------|-------------|
| SRG-APP-001035-CTR-000323 | Added | CAT I | Platform must be vendor-supported version |
| SRG-APP-000033-CTR-000090 | Modified | CAT II | Title/Fix updated |
| SRG-APP-000033-CTR-000095 | Modified | CAT II | Title/Fix updated |
| SRG-APP-000033-CTR-000100 | Modified | CAT II | Title/Fix updated |
| SRG-APP-000456-CTR-001125 | Modified | CAT II | Title/Fix updated |
| SRG-APP-000456-CTR-001130 | Modified | CAT II | Title/Fix updated |
| SRG-APP-000514-CTR-001315 | Modified | CAT II | Title/Check/Fix updated |

Net change: 187 → 188 rules (1 CAT I added for version support).

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.

## Author Information

**Alex Ackerman**

- GitHub: [@darkhonor](https://github.com/darkhonor)
- Galaxy: [darkhonor](https://galaxy.ansible.com/darkhonor)

---

*This role is provided as-is for educational and operational purposes. Always
test in a non-production environment before deploying to production systems.
Ensure compliance with your organization's security policies and change
management procedures.*
