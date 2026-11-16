# Contributing to ansible-role-rhel-rke2-stig

Thank you for your interest in contributing to this STIG compliance role!

## Code of Conduct

Please be respectful and constructive in all interactions.

## Development Environment Setup

### RHEL 9 Python Configuration

RHEL 9 system tools (like `subscription-manager`, `dnf`, etc.) require Python 3.9.
However, `ansible-core` 2.21+ requires Python 3.12+. To avoid breaking system tools,
we use a dual-Python setup:

| Command    | Version | Purpose                                    |
|------------|---------|------------------------------------------- |
| `python3`  | 3.9.x   | System tools (subscription-manager, dnf)  |
| `python`   | 3.12.x  | Development tools (ansible, pre-commit)   |

#### Installing Python 3.12 on RHEL 9

```bash
# Python 3.12 ships in the RHEL 9.4+ AppStream repository (no EPEL required)
sudo dnf install -y python3.12 python3.12-pip python3.12-devel

# Configure alternatives (keep python3 pointing to 3.9 for system tools)
sudo alternatives --install /usr/bin/python3 python3 /usr/bin/python3.9 3
sudo alternatives --set python3 /usr/bin/python3.9

# Set python (without the 3) to use 3.12 for development
sudo alternatives --install /usr/bin/python python /usr/bin/python3.12 2
sudo alternatives --set python /usr/bin/python3.12

# Verify configuration
python3 --version   # Should show Python 3.9.x
python --version    # Should show Python 3.12.x
```

> **WARNING**: Do NOT change `python3` to point to Python 3.12. This will break
> `subscription-manager`, `dnf`, and other RHEL system tools. Recovery requires
> manually creating a temporary repo with entitlement certificates to reinstall
> `subscription-manager`. Don't ask how we know this.

#### Installing Development Dependencies

All Python dependencies are hash-pinned in `requirements.txt`, which is generated
from `requirements.in` via `pip-compile --generate-hashes` (pip-tools) for
reproducible, integrity-verified installs across local development and CI/CD. Edit
`requirements.in` (not `requirements.txt`) and recompile to change a dependency.
Dependabot maintains the lock automatically.
Choose **one** of the following installation methods:

##### Option A: System-wide Installation (Simpler)

Install tools directly using the `python` (3.12) command:

```bash
# Upgrade pip first
python -m pip install --upgrade pip

# Install all dev dependencies from requirements.txt
python -m pip install --require-hashes -r requirements.txt

# Install the Ansible collections the role and its lint run need
ansible-galaxy collection install -r requirements.yml
```

##### Option B: Virtual Environment (Isolated)

Use a venv to isolate development dependencies from the system:

```bash
# Create a virtual environment with Python 3.12
python3.12 -m venv ~/.venv/ansible-dev

# Add activation alias to your shell (optional convenience)
echo 'alias ansible-dev="source ~/.venv/ansible-dev/bin/activate"' >> ~/.bashrc
source ~/.bashrc

# Activate the virtual environment
source ~/.venv/ansible-dev/bin/activate
# Or use the alias: ansible-dev

# Install dependencies inside the venv
pip install --upgrade pip
pip install --require-hashes -r requirements.txt

# Install the Ansible collections the role and its lint run need
ansible-galaxy collection install -r requirements.yml

# Deactivate when done
deactivate
```

> **Note**: When using a venv, you must activate it before running any development
> commands (ansible-lint, molecule, pre-commit, etc.).

##### Verify Installation

Regardless of which method you chose:

```bash
ansible --version      # Should show Python 3.12.x
ansible-lint --version
yamllint --version
pre-commit --version
molecule --version
```

### Pre-commit Hooks

This repository uses pre-commit hooks to ensure code quality before commits:

```bash
# Install hooks with the requirements.txt toolchain (see below)
python -m pre_commit install

# Run manually on all files
pre-commit run --all-files

# Hooks will run automatically on git commit
```

The following hooks are configured:
- **yamllint** - YAML syntax and style validation
- **ansible-lint** - Ansible best practices (production profile)
- **trailing-whitespace** - Remove trailing whitespace
- **end-of-file-fixer** - Ensure files end with newline
- **check-yaml** - Validate YAML syntax
- **check-added-large-files** - Reject any file over 500 KB, new or
  existing (`--enforce-all`)
- **check-merge-conflict** - Reject conflict markers in any file, even
  outside a merge (`--assume-in-merge`; it matches the standard Git marker
  lines, so a Markdown heading underlined with exactly `=======` is also
  rejected)
- **detect-private-key** - Prevent accidental key commits

The CI lint job runs these hooks with `pre-commit run --all-files`, plus
actionlint on the workflows. At commit time ansible-lint scans the whole
project, but the other hooks check only the files the commit stages, so run
`pre-commit run --all-files` before pushing to match the lint job.

#### Every hook uses your installed toolchain

Every hook is a `repo: local` hook. pre-commit does not build an environment or
download anything for them (ansible-lint runs `offline` and uses the collections
installed from `requirements.yml`); each runs the matching command (`yamllint`,
`ansible-lint`, `trailing-whitespace-fixer`, and the other `pre-commit-hooks`
commands) that is first on your `PATH`. Those are the versions you installed
from the hash-pinned `requirements.txt`, which are the same versions CI runs, so
a rule that fails in CI also fails your commit for the files it stages, and
there is no second copy of any version in `.pre-commit-config.yaml` to keep in
sync. For Python tooling, Dependabot updates `requirements.in` and
`requirements.txt` only. The ansible-lint hook runs on every commit, including
commits that only change the lock.

This means:

- **Install the dependencies before running pre-commit.** Use Option A or Option
  B above, including the `ansible-galaxy collection install` step. With a venv,
  activate it before `git commit`. Commits made from an IDE or GUI Git client
  use that tool's environment, which may not include the venv.
- **Run `pre-commit install` with the locked pre-commit.** The config requires
  pre-commit 4.5.0 or later. `pre-commit install` records the Python
  interpreter that ran it, and every later `git commit` uses that interpreter's
  pre-commit, even with the venv active. Run `python -m pre_commit install`
  (with the venv active, for Option B) so the locked pre-commit is recorded.
  If you installed the hooks earlier with a distro or pipx pre-commit, run it
  again.
- **Reinstall after the lock changes.** When you pull a change to
  `requirements.txt`, such as a merged Dependabot update, rerun:

  ```bash
  python -m pip install --require-hashes -r requirements.txt
  ```

  When `requirements.yml` changes, also rerun
  `ansible-galaxy collection install -r requirements.yml`. Until you do, the
  hooks run the previous versions and can disagree with CI.
- **Check the versions match.** `ansible-lint --version` reports the
  ansible-lint and ansible-core versions in use, `yamllint --version` the
  yamllint version, and `python -m pip show pre-commit-hooks` the hygiene
  hooks' version. They should equal the pins in `requirements.txt`.
- **When `pre-commit-hooks` is bumped,** the local hook definitions in
  `.pre-commit-config.yaml` are a copy of upstream's `.pre-commit-hooks.yaml`
  for the pinned version. Compare them with the new version's file in the same
  pull request and carry over any changed `entry`, `types`, or `stages`.

Troubleshooting:

- ``Executable `<command>` not found``: the dependencies are not installed or
  the venv is not active.
- ansible-lint warnings about `resolved_fqcn`: the `requirements.yml`
  collections are not installed. Run the `ansible-galaxy collection install`
  step above.
- `pre-commit version 4.5.0 is required but version X is installed`: the git
  hook was installed by an older pre-commit. Rerun `pre-commit install` as
  described above.
- A traceback, or results that differ from CI: another copy of the tool
  earlier on your `PATH` (for example a pipx or distro copy) is being used.
  Check that `command -v ansible-lint` (or the failing command) points into
  your venv or Option A install, and that the version check above matches
  `requirements.txt`.

### Molecule Testing

Molecule tests require Podman (not Docker) on RHEL:

```bash
# Install Podman
sudo dnf install -y podman

# Run full test suite
molecule test

# Development workflow (faster iteration)
molecule converge   # Create and apply role
molecule verify     # Run verification tests
molecule destroy    # Clean up
```

## How to Contribute

### Reporting Issues

- Check existing issues before creating a new one
- Include RHEL version, Ansible version, and relevant STIG IDs
- Provide steps to reproduce the issue

### Submitting Changes

1. **Fork** the repository
2. **Create a branch** for your changes:
   ```bash
   git checkout -b feature/RHEL-09-XXXXXX
   ```
3. **Follow the code style** documented in CLAUDE.md:
   - Comments wrap at 80 characters
   - Use FQCN for all modules (`ansible.builtin.*`)
   - Include STIG comment blocks for all tasks
4. **Test your changes**:
   ```bash
   pre-commit run --all-files
   molecule test
   ```
5. **Commit** with a descriptive message:
   ```bash
   git commit -m "Add RHEL-09-XXXXXX: Brief description"
   ```
6. **Push** and open a Pull Request

### Task Structure

All tasks must follow this format:

```yaml
###########################################################
# STIG ID: RHEL-09-XXXXXX    Group ID: V-XXXXXX
# Rule Title: Full title from STIG (wrap at 80 chars with
#   continuation on next line)
# Severity: CAT I | CAT II | CAT III
###########################################################
- name: RHEL-09-XXXXXX | Brief description
  ansible.builtin.module_name:
    param: value
  tags:
    - RHEL-09-XXXXXX
    - CAT II
```

### Adding New STIG Controls

1. Reference the authoritative XCCDF file for accurate metadata
2. Add variables to `vars/main.yml` if using templates
3. Update the README.md STIG coverage tables
4. Include verification in `molecule/default/verify.yml`

## Questions?

Open an issue with the `question` label.
