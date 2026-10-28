# remote_desktop

An Ansible role that sets up remote login with GNOME Remote Desktop. It
configures the gnome-remote-desktop system daemon to accept RDP clients that
authenticate with one RDP user name and password, and to serve remote login
through GDM.

## Platforms and prerequisites

The role supports SLES 16, openSUSE Leap 16 and Fedora 44 managed nodes. On
each it supports the major version of gnome-remote-desktop that the platform
ships: 48 on SLES and openSUSE Leap, and 50 on Fedora 44. The control node
needs ansible-core 2.18 or later.

- The role runs as root, so a play that connects as another user needs
  `become: true`.
- It installs packages from the repositories the host already has, so a SLES
  host must be registered.
- Users sign in at the GDM login screen with an account that has a password.
  On SLES cloud images, the default user that cloud-init creates has none.
- On a system it cannot manage, the role stops with a message before it
  changes anything.
- On openSUSE Leap 16 and Fedora 44, GNOME suspends a physical host after 15
  idle minutes, also at the login screen, and remote login is then unavailable
  until the host wakes. SLES sets no idle suspend. The role does not change
  power settings.

### Collection requirements

The role needs the collections listed in `meta/collection-requirements.yml`.
From a git checkout of the role, install them on the *control node* before
using the role:

```bash
ansible-galaxy collection install -vv -r meta/collection-requirements.yml
```

## What the role changes

- It installs `gnome-remote-desktop`, `openssl`, GDM and a GNOME session:
  `patterns-gnome-gnome` and `gnome-console` on SLES and openSUSE Leap, and
  `gdm`, `gnome-shell`, `gnome-session-wayland-session` and `ptyxis` on Fedora.
- It makes `graphical.target` the default boot target and enables and starts
  `gdm.service` at once. No reboot is needed.
- It enables and starts `gnome-remote-desktop.service`. The daemon listens on
  TCP port 3389 on all interfaces.
- It manages `/etc/gnome-remote-desktop/grd.conf` and removes the override that
  GNOME Settings writes,
  `/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/grd.conf`.
  Settings changed in the Remote Login panel of GNOME Settings or with `grdctl`
  are replaced whenever the role configures the host.
- It restarts the daemon when its settings, certificate or credentials change.
- The daemon serves remote login through GDM, so GDM's login rules decide
  which accounts can sign in, as they do for local login.

## Credentials

The RDP user name and password are one pair that every RDP client sends before
the GDM login screen appears. They are an RDP credential, not a system account.

The daemon stores them in plain text in
`/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/credentials.ini`,
with mode 0600, readable by root and by the `gnome-remote-desktop` service
account. The role passes the password to `grdctl` on standard input, never on
a command line. While gnome-remote-desktop is installed, disabling remote login
with the role clears the stored pair.

## TLS certificate

Without `remote_desktop_cert` and `remote_desktop_private_key`, the daemon
serves a self-signed certificate that the role generates as the
`gnome-remote-desktop` service account:

- the certificate is
  `/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.crt`
  and its key is
  `/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.key`
- the key is RSA 4096, and the certificate is valid for 720 days
- the common name is the host's short name (at most 64 characters), and the
  subject alternative names are the FQDN and the short name

A run replaces the certificate when it is missing or invalid, or when it
expires within 30 days. A new certificate has a new fingerprint.

A supplied pair is checked with openssl before any configuration change. The
service account then copies it to
`/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls-supplied.crt`
and
`/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls-supplied.key`,
with mode 0600. The daemon reads its certificate only when it starts, so a
replaced source file takes effect at the next run of the role, which copies it
and restarts the daemon. While gnome-remote-desktop is installed, disabling
remote login with the role removes the copies and keeps the self-signed pair.

## Role Variables

Validation is enforced by `meta/argument_specs.yml`.

### remote_desktop_enabled

Turns remote login on (`true`) or off (`false`).

With `true`, the role installs, configures, starts and verifies remote login.
Credentials are the opt-in: without `remote_desktop_username` and
`remote_desktop_password`, a run continues only when remote login is already
on, that is when `gnome-remote-desktop.service` is enabled, its settings say
`enabled=true` and a credential pair is stored. Otherwise the role stops
before it changes anything, with a message that names both variables.

With `false`, the role writes `enabled=false`, clears the stored credentials,
removes the copies of a supplied certificate and key, and stops and disables
`gnome-remote-desktop.service`. With `remote_desktop_manage_firewall: true`,
it also removes the firewall entry, as described there. It installs nothing,
ignores the credential variables, and keeps the packages, GDM, the default boot
target and the self-signed certificate. When gnome-remote-desktop is not
installed, it changes only the firewall entry. The default value is `true`.

```yaml
remote_desktop_enabled: false
```

### remote_desktop_username

The user name that RDP clients send. It is an RDP credential, not a system
account, and is set together with `remote_desktop_password`. When both are
null or empty, the stored pair is kept, but only while remote login is already
on, and the run says that it kept the pair.

A user name has no leading or trailing whitespace, no control characters or
line separators, no `:` and no `\`, does not start with `-`, and is not the
string `None`, which a prompt returns when there is no terminal. The default
value is `null`.

```yaml
remote_desktop_username: rdpuser
```

### remote_desktop_password

The password that RDP clients send. Keep it in Ansible Vault or in a file that
only its owner can read.

A password has at least 15 characters, has no leading or trailing whitespace
and no control characters or line separators, and does not contain the user
name or the host's short name, in any letter case, when that name has 4 or
more characters.

Ansible treats `{{`, `{%` and `{#` in a variable as the start of a template, so
a password that contains them must be tagged `!unsafe` in the vars or vault
file. Otherwise Ansible tries to render it, and the error can show the
password. Values given on the command line as `-e name=value` appear in the
process list and in sudo's log; give the password at the prompt of the example
playbook or in a file passed with `-e @file` instead. The default value is
`null`.

```yaml
remote_desktop_password: "{{ vault_remote_desktop_password }}"
```

### remote_desktop_cert

Absolute path, on the managed node, of a PEM certificate for the daemon to
serve instead of the self-signed one. It is set together with
`remote_desktop_private_key`. Both must be regular files; a symbolic link is
refused. Before it changes any configuration, the role checks that the
certificate parses, has not expired and matches the key. When both variables
are unset again, the daemon serves the self-signed certificate and the role
removes the copies. The default value is `null`.

```yaml
remote_desktop_cert: /etc/ssl/rdp/server.crt
```

### remote_desktop_private_key

Absolute path, on the managed node, of the unencrypted PEM private key for
`remote_desktop_cert`. An encrypted key is refused, because the daemon has no
setting for a passphrase. The key never leaves the managed node and never
appears in the role's output. The default value is `null`.

```yaml
remote_desktop_private_key: /etc/ssl/rdp/server.key
```

### remote_desktop_manage_firewall

Opens TCP port 3389, the firewalld service `rdp`, in the default zone for all
sources, permanently and at runtime. The role does this through the firewall
system role, `fedora.linux_system_roles.firewall`, and only when firewalld is
installed, enabled and running. Otherwise it changes nothing and says that no
host firewall rule was added; a firewalld that is enabled but stopped starts
later with port 3389 closed. Unlike other system roles that manage the
firewall, this role never installs, unmasks, enables or starts firewalld.

The entry is added after the daemon is verified to be listening. With
`remote_desktop_enabled: false`, the entry is removed only when this role had
set up remote login on the host, shown by its marker line in
`/etc/gnome-remote-desktop/grd.conf` at the start of the run, and no other
program listens on port 3389. The default value is `false`.

```yaml
remote_desktop_manage_firewall: true
```

### remote_desktop_secure_logging

If `true`, suppress potentially sensitive output from tasks that handle
credentials, secrets, and other sensitive data by setting `no_log: true` on
those tasks. This prevents passwords, API tokens, private keys, and similar
sensitive information from appearing in Ansible logs and console output.

If you need to debug issues with credential handling or secret management, you
can temporarily set `remote_desktop_secure_logging: false` to see the full
output from these tasks. However, be aware that this may expose sensitive
information in logs, so it should only be used in development or
troubleshooting scenarios. With `false`, the RDP password is also written to
the system journal of the managed node. The default value is `true`.

```yaml
remote_desktop_secure_logging: true
```

### remote_desktop_transactional_update_reboot_ok

This variable is used to handle reboots required by transactional updates. If a
transactional update requires a reboot, the role will proceed with the reboot if
`remote_desktop_transactional_update_reboot_ok` is set to `true`. If set to
`false`, the role will notify the user that a reboot is required, allowing for
custom handling of the reboot requirement. With `false`, the role stops after
the notice, before it configures anything; reboot and run the role again. If
this variable is not set, the role will fail to ensure the reboot requirement
is not overlooked. For non-transactional update systems, this variable is
ignored. The default value is `null`.

```yaml
remote_desktop_transactional_update_reboot_ok: true
```

## Variables Exported by the Role

### remote_desktop_tls_fingerprint

The SHA-256 fingerprint of the certificate the daemon serves, or, when the
daemon does not report it, the fingerprint of the configured certificate.
Compare it with
the fingerprint an RDP client shows when it first connects. The role sets it
after it has verified the listener, on a booted host and outside check mode.
Any other run of the role leaves it empty.

## Example Playbook

The `ansible-linux-system-roles` package installs two example playbooks that
configure the host they run on:

- `/usr/share/ansible/collections/ansible_collections/suse/linux_system_roles/docs/remote_desktop/remote_login.yml`
  turns remote login on, opens port 3389 in a firewalld that is enabled and
  running, and prints where to connect and the certificate fingerprint.
- `/usr/share/ansible/collections/ansible_collections/suse/linux_system_roles/docs/remote_desktop/remote_login_disable.yml`
  turns remote login off.

Run them on the host itself:

```bash
sudo ansible-playbook -c local -i localhost, /usr/share/ansible/collections/ansible_collections/suse/linux_system_roles/docs/remote_desktop/remote_login.yml
```

`remote_login.yml` asks for the RDP user name, the password, and the password
again. Empty answers keep the stored pair while remote login is on. Variables
given with `-e @file` are not asked for.

To set up several hosts, keep the credentials in a vars file encrypted with
Ansible Vault that sets `remote_desktop_username` and
`remote_desktop_password`:

```yaml
- name: Set up remote login over RDP
  hosts: all
  become: true
  vars_files:
    - rdp-credentials.yml
  vars:
    remote_desktop_manage_firewall: true
  roles:
    - role: suse.linux_system_roles.remote_desktop
```

## License

MIT

## Author Information

Maintained by SUSE (Harshvardhan Sharma, <harshvardhan.sharma@suse.com>).
