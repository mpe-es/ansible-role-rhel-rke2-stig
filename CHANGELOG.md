# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed

- Bumped molecule to 26.9.0 and molecule-plugins[podman] to 26.9.28. Both now
  validate `platforms[].tmpfs` as a list, while `containers.podman.podman_container`
  requires a dict, so the Molecule platform drops its `tmpfs:` mounts and pins
  podman systemd mode (`systemd: "always"`), which mounts `/run` and `/tmp` itself.
  The Molecule container's `/run` and `/tmp` are no longer `noexec`; Molecule does
  not reproduce STIG `/tmp` partitioning. The CI podman pre-flight now mirrors this
  config and fails if systemd does not reach `running` or `degraded`.
- Pinned `containers.podman` 1.21.1 in `molecule/default/requirements.yml`, and
  pointed the Molecule galaxy dependency at that scenario file via
  `${MOLECULE_SCENARIO_DIRECTORY}`. The bare `requirements.yml` resolved against
  the working directory, so standalone `molecule test` runs installed the root
  file's `>=` ranges instead of the scenario's exact pins.

## [0.4.1] - 2026-08-23

### Fixed

- **Audit compliance did not survive a reboot.** A node hardened to 202 of 207 rules with
  `enabled 2` came back from any restart with ~20 rules and audit **MUTABLE**, silently,
  and stayed that way until the role ran again - both states reporting success. `/run` is
  tmpfs, so `/run/k3s/containerd/containerd.sock` does not exist until RKE2 starts, and
  `audit-rules.service` runs at boot **before** rke2-server/rke2-agent. The watch failed
  with `No such file or directory` and `augenrules` aborted the load at that line, dropping
  every later rule and the trailing `-e 2`. The handler's tolerance for that error - correct
  for build-time pre-staging - is what kept it quiet, and the immutability check from #67
  could not catch it because the node never reaches `-e 2` at boot to have a pending change
  detected. The watch is removed; verified across real reboots of both an agent and the
  control plane, each returning `201/206` rules with `enabled 2`. Other watches on paths
  absent at boot (the server-only paths on an agent) do not abort the load, so no post-RKE2
  reload unit is needed. (#74)

- The audit rule load still aborted after v0.4.0. That release removed four rules the
  SSG/SCAP base profile already provides, based on a measurement that had **missed four
  more** because the comparison did not normalise `-F` field ordering. On a live cluster
  v0.4.0 reached **129 of 205 rules with audit still MUTABLE** - a convincing-looking
  improvement over v0.3.2's 8, and still broken. There were 8 collisions, not 4:
  `chmod`, `chown` and `xattr` against `perm_mod.rules`, plus `delete`, `export` and
  `logins` against their own base files. (#66)
- Replaced that static removal with a **runtime delegation** step, and restored the four
  rules v0.4.0 deleted so the dataset is complete again and every STIG ID stays traceable.
  At apply time the role compares its rendered ruleset against the other files in
  `rules.d` and comments out only exact matches, recording the `file:line` each rule
  defers to. Normalisation handles all three equivalent encodings: `-S a,b` vs
  `-S a -S b`, `-k KEY` vs `-F key=KEY`, and `-F` field order.
  On a host **without** the base profile nothing matches, nothing is commented, and the
  role fills the gap exactly as documented - verified as a no-op in that case. A rule the
  base profile later stops providing is automatically restored rather than staying
  suppressed. Measured on a live cluster: **12 rules delegated, 202 of 207 loaded,
  `auditctl -s` reporting `enabled 2`**.
- `audit-rules.service` kept its boot-time `failed` state even after the role successfully
  reloaded the ruleset, because systemd holds a failed unit until it is explicitly reset.
  The unit is now reset **only when the reload actually succeeded**, so `systemctl --failed`
  stops accusing a node the role just fixed.

### Changed

- The audit rules template now renders to a candidate under `/opt/rke2-stig/` and the
  delegation step writes `/etc/audit/rules.d/20-stig.rules`, replacing it only when the
  result differs. Rendering straight to the live path wiped the delegation markers on
  every run, reported changed, notified the reload handler, and on a node that had since
  reached `-e 2` that was a **permanent failure** - the play never converged. Verified
  idempotent both in Molecule and against a live cluster.

## [0.4.0] - 2026-08-23

### Added

- Audit rule load verification. `augenrules --load` returns 0 even when it applies
  nothing, so a ruleset that the kernel rejected mid-load left the play reporting
  success while most of the policy was never loaded. The reload handler now reads
  immutability *before* reloading, fails when rule changes could not be applied
  because the audit system is already immutable (`-e 2`, new opt-out
  `rhel_rke2_stig_audit_immutable_pending_is_fatal`), and asserts that a ruleset
  ending in `-e 2` actually reached `enabled 2` — a single high-signal check that
  the load ran to completion. (#67)
- AIDE exclusions for Kubernetes container logs (`/var/log/pods`,
  `/var/log/containers`) via `rhel_rke2_stig_aide_exclusions`, with
  `rhel_rke2_stig_aide_exclusions_enabled` (default `true`) and
  `rhel_rke2_stig_aide_conf_path` (default `/etc/aide.conf`). Stock `aide.conf`
  watches `/var/log` recursively, kubelet creates those paths after image bake, and
  `aide_use_fips_hashes` puts `sha512` into the size-only `LOG` group — so
  append-only container logs are content-hashed and `aide --check` can never return
  0 on a running node. Emits an explicit warning that the AIDE database must be
  re-initialized, which is a deploy-time action this role deliberately does not
  perform. (#65)
- RHEL 10 STIG (V1R1) support via version-gated dataset loading: RHEL 10 audit
  rules, kernel module blacklist, and RHEL 10-gated `all.rp_filter` / IPv6
  forwarding toggles.
- Renovate configuration to keep the Rocky base-image digest current
  (digest-only; the pins Dependabot cannot track). (#36, #40)
- Supply-chain cooldown window for Dependabot updates, standardized with the
  sibling repositories. (#35)
- Dependabot `codeql-action` version-lockstep group, so the `github/codeql-action`
  sub-actions are always proposed as one PR at a single SHA instead of drifting
  apart into a broken analysis.
- `actionlint` enforced in the Lint job, so workflow syntax errors fail CI instead
  of surfacing only at run time. (#46)
- Community health files (Code of Conduct, Contributing, Security policy), issue
  and pull request templates, and CodeQL code scanning. (#42)

### Changed

- Molecule verify now derives the expected audit rule count from the dataset instead
  of asserting a hardcoded `>= 50` floor. The template emits one `STIG-ID:` comment
  per entry in `rhel_rke2_stig_audit_rules`, so the check asserts an exact match —
  which also catches a truncated or partially rendered template, something a floor
  cannot. The magic number had broken when four duplicate rules were correctly
  removed, and would have broken again on any legitimate dataset change. (#69)
- Documented, without changing behaviour, why an OpenSCAP STIG-profile scan reports
  `file_permissions_cron_d`/`_daily`/`_hourly`/`_weekly`/`_monthly` and
  `file_permissions_crontab` as failed on a **compliant** node. DISA
  `RHEL-09-232040` requires cron permissions "not be modified from the operating
  system defaults" — 0755 directories and 0644 `/etc/crontab`, which
  `rpm --setperms` restores. SSG's `file_permissions_cron_*` rules demand
  0700/0600 but carry no `RHEL-10-nnnnnn` DISA rule ID, only the generic
  `SRG-OS-000480-GPOS-00227` plus CIS/NIST references, and the RHEL 10 datastream
  contains no STIG-ID-mapped cron permission control at all. Tightening these would
  satisfy the scanner and violate the STIG. Recorded in `tasks/main.yml` and the
  README so the divergence is not "fixed" by a future reader. (#68)

### Removed

- Dead `cgroupns_mode: host` key from the Molecule platform config. It is not a
  supported molecule podman-driver option and was never forwarded to podman in
  any shipped version, so it was a silent no-op behind a comment claiming
  cgroups v2 compatibility for GitHub Actions runners, an assurance the key
  never actually provided. systemd-as-PID-1 works via `privileged`, the `/run`
  tmpfs mount, and `SYS_ADMIN`.

### Fixed

- Removed the audit rule targeting `/var/lib/rancher/rke2/bin/crictl`. That path
  traverses a symlink into RKE2's versioned data directory, and the audit kernel
  does not resolve symlinks for `-F path=` watches — it rejects the rule with
  `EPERM`, and `augenrules` aborts the entire load at that line. On a live cluster
  this left **8 of 205 rules loaded** with the audit configuration still mutable.
  No replacement watch was added: a `-p x` watch on the data directory would record
  every containerd/runc/kubelet exec, and the SSG base sets `-f 2` (panic on audit
  failure), so flooding the backlog is a kernel panic rather than lost records. The
  directory is `0750 root:root`, so `auid>=1000` execution was already impossible
  rather than merely unaudited. (#64)
- Removed four audit rules that duplicate the SSG/SCAP `perm_mod.rules` the base
  STIG profile already applies (`chown` family and the `auid>=1000` `xattr` family,
  b32 and b64). They are identical to the kernel but textually different, and
  `augenrules` performs no deduplication whatsoever — it emits every rule verbatim
  — so the kernel rejected the second copy with `Rule exists` and aborted the load,
  stopping at **129 of 205 rules** with `-e 2` never reached. Re-encoding to match
  SSG cannot help, precisely because there is no dedupe. The `auid=0` `xattr` pair
  is not emitted by the base profile and is retained. (#66)
- Re-pinned the Rocky Linux 9 UBI-init image to a live digest after the pinned
  digest was removed upstream, restoring CI across all branches. (#34)
- Dropped a trailing slash on the RHEL 10 `/etc/sudoers.d` audit path that made
  the rule kernel-unloadable.
- Tolerate missing RKE2 audit watch paths when pre-staging on a golden image.
- Restore the CodeQL workflow by moving `github/codeql-action` `init`,
  `analyze`, and `upload-sarif` to a single release (v4.37.6) in one commit.
  The sub-actions share one monorepo commit and the action fails closed when
  they disagree, so the per-sub-action Dependabot bumps (#50, #51, #53) each
  broke the analysis on their own and could not be fixed by merging them
  individually, because they targeted different releases.

### Security

- Gate the `CI Status` check on the `security` job. `security` was listed in
  the check's `needs` but omitted from its failure condition, so a Trivy
  CRITICAL or a CodeQL finding failed the Security Scan job while the single
  required status context for branch protection still reported success and the
  pull request stayed mergeable.
- Set explicit least-privilege `permissions` on all CI and release workflow
  jobs, resolving the CodeQL `actions/missing-workflow-permissions` findings.
- Update `cryptography` to 48.0.1, remediating GHSA-537c-gmf6-5ccf (vulnerable
  OpenSSL bundled in the cryptography wheels). (#43)

### Dependencies

- Bump ansible-core from 2.21.0 to 2.21.1
- Bump ansible-lint from 26.4.0 to 26.6.0 (#31)
- Bump molecule from 26.4.0 to 26.6.0 (#30)
- Bump molecule-plugins from 25.8.12 to 26.7.8 (#33)
- Bump actions/checkout from 6.0.3 to 7.0.0
- Bump actions/setup-python from 6.2.0 to 6.3.0 (#28)
- Bump github/codeql-action from 4.36.1 to 4.36.2
- Bump github/codeql-action/upload-sarif from 4.36.2 to 4.37.0 (#32)
- Bump cryptography from 48.0.0 to 48.0.1 (#43)
- Bump ansible-core from 2.21.1 to 2.21.2 (#47)
- Bump pre-commit from 4.6.0 to 4.6.1 (#48)
- Bump actions/checkout from 7.0.0 to 7.0.1 (#52)
- Bump actions/setup-python from 6.3.0 to 7.0.0 (#54)
- Bump cryptography from 48.0.1 to 50.0.0 (#55)
- Bump pre-commit from 4.6.1 to 4.6.2 (#59)
- Bump ansible-core from 2.21.2 to 2.21.3 (#60)
- Bump ansible-lint from 26.6.0 to 26.8.0 (#62)
- Bump reviewdog/action-actionlint from 1.72.0 to 1.73.1 (#57)
- Bump github/codeql-action from 4.37.0 to 4.37.6 (`init`, `analyze`,
  `upload-sarif` in lockstep)

## [0.3.2] - 2026-06-04

Maintenance release: STIG baseline refresh and Galaxy namespace alignment. No
changes to role remediation behavior since v0.3.1.

### Changed

- Updated to RHEL 9 STIG V2R8 and RKE2 STIG V2R6 (Container Platform SRG remains
  V2R4).
- Reclassified three FIPS controls (RHEL-09-215105, -255070, -255075) from
  CAT II to CAT I per DISA; severity metadata and docs updated.
- Role namespace updated to `mpe_es` to match the canonical organization repo.

### Fixed

- Corrected RHEL-09-215105 (V-258241): COMPLIANT via FIPS:AD-SUPPORT, removed
  from the STIG exemptions list.

## [0.3.1] - 2026-06-04

Maintenance and supply-chain hardening release. No changes to role behavior
since v0.3.0.

### Changed

- Replaced the Molecule test base with Rocky Linux 9 (ubi-init), removing an
  external-egress dependency and improving STIG fidelity.
- CI Python updated to 3.12 to match ansible-core 2.21.

### Security

- SHA-pin all GitHub Actions; digest-pin the Molecule image; hash-pin the Python
  toolchain (`--require-hashes`); freeze pre-commit hook revisions to SHAs.

### Dependencies

- ansible-core 2.21.0, ansible-lint 26.4.0, molecule 26.4.0,
  molecule-plugins[podman] 25.8.12, yamllint 1.38.0, pre-commit 4.6.0,
  jmespath 1.1.0.

## [0.3.0] - 2026-02-06

### Added

- Kernel hardening sysctl settings; SSH MACs, ClientAlive, pubkey auth, and
  crypto policy; PAM hashing, smart card, and SSSD; file permissions and mail
  relay; NetworkManager DNS; RKE2 operational configs and exemption docs.

### Changed

- Updated STIG baseline to V2R7; made the user-namespace limit configurable.

### Fixed

- Write IP forwarding settings to `/etc/sysctl.conf`; container-tolerant sysctl
  and handler reloads; install `community.general` for the syntax check.

## [0.2.0] - 2026-01-23

### Changed

- Molecule testing moved to UBI9-init with CentOS repos (SCRM-compliant), with
  cgroups v2 fixes for GitHub Actions runners.

### Fixed

- Corrected `PASS_MIN_DAYS` verification logic; made auditd reload
  container-aware.

## [0.1.0] - 2026-01-23

### Added

- Initial RHEL 9 / RKE2 STIG remediation role: baseline controls, Molecule +
  Podman CI, pre-commit hooks, and pinned Python requirements.

[Unreleased]: https://github.com/mpe-es/ansible-role-rhel-rke2-stig/compare/v0.4.1...HEAD
[0.4.1]: https://github.com/mpe-es/ansible-role-rhel-rke2-stig/compare/v0.4.0...v0.4.1
[0.4.0]: https://github.com/mpe-es/ansible-role-rhel-rke2-stig/compare/v0.3.2...v0.4.0
[0.3.2]: https://github.com/mpe-es/ansible-role-rhel-rke2-stig/compare/v0.3.1...v0.3.2
[0.3.1]: https://github.com/mpe-es/ansible-role-rhel-rke2-stig/compare/v0.3.0...v0.3.1
[0.3.0]: https://github.com/mpe-es/ansible-role-rhel-rke2-stig/compare/v0.2.0...v0.3.0
[0.2.0]: https://github.com/mpe-es/ansible-role-rhel-rke2-stig/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/mpe-es/ansible-role-rhel-rke2-stig/releases/tag/v0.1.0
