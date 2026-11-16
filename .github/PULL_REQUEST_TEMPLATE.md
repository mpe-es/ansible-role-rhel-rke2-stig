<!--
  Thank you for contributing to ansible-role-rhel-rke2-stig.
  Please complete the sections below. See CONTRIBUTING.md for full guidance.
-->

## Summary

<!-- What does this change do, and why? -->

## Type of change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New STIG control coverage or remediation
- [ ] Enhancement to existing behavior
- [ ] CI / tooling / supply-chain
- [ ] Documentation
- [ ] Breaking change (existing behavior changes for consumers)

## Related issues

<!-- e.g. "Closes #123". Use "Refs #123" for partial work. -->

## STIG / SRG references

<!--
  If this touches remediation, list the affected control or rule IDs
  (e.g. RHEL-09-215105 / V-258241, RKE2 rule IDs) and the baseline version.
  Write "N/A" if not applicable.
-->

## Checklist

- [ ] Commits follow Conventional Commits (`feat:`, `fix:`, `docs:`, `ci:`, ...)
- [ ] `pre-commit run --all-files` passes locally (includes yamllint and ansible-lint)
- [ ] `molecule test` passes for affected scenarios
- [ ] Playbook `--syntax-check` passes
- [ ] Documentation (README / CONTRIBUTING / defaults) updated as needed
- [ ] `CHANGELOG.md` `[Unreleased]` section updated
- [ ] No secrets, credentials, or operational identifiers are included
- [ ] This PR does not report a security vulnerability (use private reporting instead)

<!--
  Do not report security vulnerabilities in a public pull request.
  See SECURITY.md for private reporting.
-->
