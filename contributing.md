# Contributing to the remote_desktop role

## Where to start

The role follows the conventions of the Linux System Roles project. Their
[Contribute](https://linux-system-roles.github.io/contribute.html) page has all
of the common information that role developers need:

* Role structure and layout
* Development tools - How to run tests and checks
* Ansible recommended practices
* Basic git and github information
* How to create git commits and submit pull requests

**Code** is managed on [GitHub](https://github.com/SUSE/ansible-remote_desktop), using
[Pull Requests](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests).

## GitHub CI testing using /citest

GitHub CI tests do not run automatically on pull requests. A role repository
maintainer must start them by posting a `/citest` slash command as a pull
request comment.

Run every available CI workflow:

```text
/citest all
```

Run the linting and other lightweight checks:

```text
/citest linters
```

Run the integration tests in QEMU virtual machines:

```text
/citest integration
```

Run one or more selected workflows by separating their names with spaces:

```text
/citest ansible-lint
/citest ansible-lint markdownlint
```

Post another `/citest` comment at any time to run another selection.

The table below lists each command, the check name shown in the pull request
checks list, and what the test does. Matrix jobs produce one check per
combination; those rows show the check name pattern.

| Command | Check name | Description |
| --- | --- | --- |
| `/citest all` | All checks listed below | Run every CI test available for this role |
| `/citest linters` | Lint and lightweight checks | Run ansible-lint, ansible-test, ansible-managed-var-comment, codespell, markdownlint, pr-title-lint, and test_converting_readme |
| `/citest integration` | QEMU checks | Run qemu-kvm-integration-tests |
| `/citest ansible-lint` | `Ansible Lint / ansible_lint (<ansible-lint>, <ansible>, <python>) (pull_request)` | Lint Ansible content after converting the role to collection format |
| `/citest ansible-managed-var-comment` | `Check for ansible_managed variable use in comments / ansible_managed_var_comment (pull_request)` | Fail if `ansible_managed` is used in comments |
| `/citest ansible-test` | `Ansible Test / ansible_test (<ansible>, <python>) (pull_request)` | Run ansible-test sanity tests |
| `/citest codespell` | `Codespell / Check for spelling errors (pull_request)` | Check for spelling errors |
| `/citest markdownlint` | `Markdown Lint / markdownlint (pull_request)` | Lint Markdown files |
| `/citest pr-title-lint` | `PR Title Lint / commit-checks` | Check that the pull request title follows the required format |
| `/citest qemu-kvm-integration-tests` | `Test / scenario (<image>, <env>) (pull_request)` | Run role integration tests in QEMU VMs on the images that match the role's galaxy tags (openSUSE Leap 16.0) |
| `/citest test_converting_readme` | `Test converting README.md to README.html / test_converting_readme (pull_request)` | Convert README.md to HTML |
| `/citest woke` | `Woke / Detect non-inclusive language (pull_request)` | Detect non-inclusive language |

## AI Coding Assistants

The `.coderabbit.yaml` configuration file in the repository root contains coding
standards and requirements that can be used by AI coding assistants to help
generate code that follows project conventions.

## README.md in the package

The `ansible-linux-system-roles` package converts README.md when it is built:

* It deletes the sections titled "Requirements", "Collection requirements" and
  "Compatibility", so nothing a package user needs goes under those headings.
* Relative links and links to anchors break in the packaged copy, so write
  installed paths out in full.
* It rewrites any string made of the role name, one character and the role
  name again.

## Running CI Tests Locally

### Use tox-lsr with qemu

tox-lsr runs the role's tests in local QEMU virtual machines. Install it and
its image configuration file as described in
<https://github.com/linux-system-roles/tox-lsr#qemu-testing>, then run a test
from a git clone of this repository, for example:

```bash
tox -e qemu-ansible-core-2-20 -- --image-name leap-16.0 tests/tests_default.yml
```

There are many command line options and environment variables which can be used
to control the behavior, and you can customize the testenv in tox.ini. See
<https://github.com/linux-system-roles/tox-lsr#qemu-testing>

See <https://linux-system-roles.github.io/contribute.html> for general
development guidelines.
