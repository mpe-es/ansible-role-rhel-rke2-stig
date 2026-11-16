# Security Policy

## Reporting a Vulnerability

The `ansible-role-rhel-rke2-stig` project takes security seriously. Because this
role applies DISA STIG hardening to RHEL 9/10 hosts and RKE2 clusters intended
for deployment in DoD enclaves, vulnerability handling is treated as a high
priority.

**Do not report security vulnerabilities through public GitHub issues, pull
requests, or discussions.**

### How to Report

Report vulnerabilities privately through GitHub's built-in **private
vulnerability reporting**:

1. Go to the repository's **Security** tab.
2. Select **Report a vulnerability** (under *Advisories*).
3. Complete the advisory form with the details below.

This opens a private security advisory visible only to the reporter and the
project maintainer ([@darkhonor](https://github.com/darkhonor)); it is never
publicly visible unless and until a fix is published. If you require encrypted
communication beyond the private advisory channel, request a public key in your
initial report and one will be provided out-of-band.

### What to Include

Please include as much of the following information as possible to help us
understand and reproduce the issue:

- The role version (release tag or git commit hash)
- The target platform (RHEL 9 or RHEL 10) and RKE2 version
- The Ansible / ansible-core version used to run the role
- A description of the vulnerability and its potential impact
- Steps to reproduce the issue
- Any proof-of-concept code or playbook (please mark clearly as such)
- Suggested mitigations or fixes if you have them

### What to Expect

You can expect the following response from the maintainer:

| Phase | Target Time |
| ----- | ----------- |
| Initial acknowledgment | Within 3 business days |
| Initial assessment and severity rating | Within 7 business days |
| Patch development for confirmed Critical/High issues | Within 30 days |
| Patch development for confirmed Medium issues | Within 90 days |
| Coordinated disclosure (if applicable) | After patch release |

For DoD enclave deployments, vulnerability information may need to be handled
under controlled channels per the relevant Information System Security Officer
(ISSO) and Authorizing Official (AO) guidance. The maintainer will coordinate
with the reporter on appropriate handling.

## Supported Versions

`ansible-role-rhel-rke2-stig` follows [Semantic Versioning](https://semver.org/).
Security fixes are applied to the most recent release.

| Version | Supported |
| ------- | --------- |
| 0.x.x | Active development |

Once the project reaches a stable 1.0.0 release, this matrix will be updated to
reflect the supported version policy.

## Security Practices

This project follows these supply chain and code security practices:

- **Dependency monitoring:** Dependabot maintains the Python toolchain (pip) and
  GitHub Actions; Renovate maintains what Dependabot cannot track: the
  container base-image digest pins and the Ansible Galaxy collection pins.
- **Supply-chain cooldown:** dependency bump PRs are held for a detection
  window (pip and Galaxy collections 5 days, majors 10 days; GitHub Actions
  5 days) so a compromised or yanked upstream release ages before adoption.
  The Rocky base-image digest is taken without a cooldown: it is a single
  curated vendor image, Trivy-scanned at PR time.
- **Pinned dependencies:** the Python toolchain is hash-pinned
  (`pip-compile --generate-hashes`, installed with `--require-hashes`) and
  every pre-commit hook runs from it; Galaxy collections are pinned to exact
  versions (Galaxy offers no hash pinning); GitHub Actions are pinned to full
  commit SHAs; the Molecule test image is digest-pinned.
- **Vulnerability scanning:** Trivy scans run in CI, with results uploaded as
  SARIF to GitHub code scanning.
- **Vulnerability alerts:** GitHub vulnerability alerts and automated security
  updates are enabled.
- **Branch hygiene:** squash-only merges, delete-branch-on-merge, and
  conventional commits.

## Dependencies and Third-Party Code

This role depends on third-party Python packages, Ansible collections, and a
container base image used only for testing. Vulnerabilities in those
dependencies are addressed by updating the relevant dependency to a patched
version. If no patched version exists, the maintainer will document the risk and
apply appropriate mitigations.
