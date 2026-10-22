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
Dependabot maintains the lock automatically for dependencies it manages.
`ansible-core` is deliberately excluded because its pin, generated lock, and
pre-commit hook must move together; maintainers update it through a coordinated
PR, including for security releases.
Choose **one** of the following installation methods:

##### Option A: System-wide Installation (Simpler)

Install tools directly using the `python` (3.12) command:

```bash
# Upgrade pip first
python -m pip install --upgrade pip

# Install all dev dependencies from requirements.txt
python -m pip install -r requirements.txt
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
pip install -r requirements.txt

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
# Install hooks (one-time setup)
pre-commit install

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
- **check-added-large-files** - Prevent large files (>500KB)
- **check-merge-conflict** - Detect merge conflict markers
- **detect-private-key** - Prevent accidental key commits

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
   yamllint .
   ansible-lint
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
