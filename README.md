# ansible-ldap

Ansible playbooks that bootstrap and manage an OpenLDAP (`slapd`) server on RHEL/CentOS-family
hosts: base directory tree, password policy, ACLs, TLS, sudo-LDAP integration, service accounts,
and user/group provisioning.

## What it does

`bootstrap-ldap.yaml` brings a bare host up to a working, secured LDAP server in one run:

1. Installs `openldap-servers`/`openldap-clients` and wipes any existing `slapd.d`/database.
2. Loads the base `cn=config` (global config, `back_mdb` module, `ppolicy`/`auditlog`/`memberof`
   overlays), the requested schemas (`core`, `cosine`, `inetorgperson`, `nis` by default), and the
   `mdb` database definition.
3. Starts and enables `slapd`, opening the `ldap`/`ldaps` firewalld services.
4. Populates the directory tree (`ou=People`, `ou=Groups`, `ou=Computers`, `ou=Policies`,
   `ou=ServiceAccounts`, a default password policy).
5. Applies TLS config if `tls_enabled` is set.
6. Creates LDAP-bind service accounts (`nss`, `sudo`, `radius`, `bind`, `admin`).
7. Applies post-config `ldapmodify` overlays — notably the locked-down production ACL set and
   the SASL/SSL `olcAuthzRegexp` mapping.

A bootstrap flag (`/etc/.openldap-installed`) makes the install idempotent — re-running
`bootstrap-ldap.yaml` will start/reconfigure but skip the destructive install steps.

Everything is driven from Jinja2-templated LDIFs under `templates/`, so the schema/ACL/policy
content lives in version control rather than in the tasks themselves.

## Requirements

- Control node: Ansible with the `community.general` and `ansible.posix` collections installed
  (a `env/` Python virtualenv with `ansible` is already set up in this repo but is git-ignored —
  recreate it with `python3 -m venv env && env/bin/pip install ansible community.general ansible.posix`
  if it's missing).
- Target hosts: RHEL/CentOS/Rocky/Alma family (`ansible_facts.os_family == "RedHat"`). The
  full-install path in `bootstrap-ldap/bootstrap-ldap-rl.yaml` only runs on RedHat-family hosts;
  Debian is not currently wired up for initial install (only `enable-sudo.yaml` has a Debian
  branch for the `sudo-ldap` package).
- Root/`become` access on the LDAP host.

## Configuration

`ansible.cfg`, `inventory.yaml`, and `vault.yaml` are git-ignored — the repo ships only
`vars.yaml` as the template of defaults. Create the other three locally before running anything:

- **`ansible.cfg`** — points at `inventory.yaml` and sets SSH options. Example:
  ```ini
  [ssh_connection]
      ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
  [defaults]
     interpreter_python = auto_silent
     inventory = inventory.yaml
  ```
- **`inventory.yaml`** — defines the `ldap_servers` group (and `ansible_user`, typically `root`).
- **`vault.yaml`** — an `ansible-vault`-encrypted file overriding secrets from `vars.yaml`
  (`admin_pw`, service account passwords, etc.). Never commit real secrets to `vars.yaml`.
- **`vars.yaml`** — everything else: `base_dn`, hostname/connection settings, service-account
  bind DNs, TLS paths, password-policy tuning (`var_max_password_age`, `var_min_password_length`,
  lockout thresholds, ...), default shell/home, the schema list, and the `var_approved_users`
  list consumed by `add-users.yaml`.

Run playbooks with `--ask-vault-pass` (or `--vault-password-file`) whenever `vault.yaml` is used.

## Playbooks

| Playbook | Purpose |
|---|---|
| `bootstrap-ldap.yaml` | Full server bootstrap (see above). Idempotent via the install flag file. |
| `enable-tls.yaml` | Re-applies just the TLS block from `bootstrap-ldap/configure-tls.yaml`. |
| `enable-sudo.yaml` | Installs the sudo LDAP client package and loads the `sudoRole` schema (`templates/other/sudo.ldif`) into `cn=config` if not already present. |
| `service-accounts.yaml` | Re-applies just the service-account creation step. |
| `ldapmodify.yaml` | Re-applies just the `templates/ldapmodify/*.ldif` overlays (ACLs, SSL authz mapping). |
| `add-users.yaml` | Runs against `localhost` (not `ldap_servers`) over `ldap://localhost:389` — creates POSIX groups/users in `ou=Groups`/`ou=People` from `var_approved_users`, wired to the shadow/password-policy attributes. Intended to be run through an SSH tunnel or from the LDAP host itself. |

Example runs:

```bash
# Full bootstrap
ansible-playbook bootstrap-ldap.yaml --ask-vault-pass

# Add/update approved users
ansible-playbook add-users.yaml --ask-vault-pass

# Add sudo-LDAP support
ansible-playbook enable-sudo.yaml --ask-vault-pass
```

## Repository layout

```
bootstrap-ldap/            # task files included by bootstrap-ldap.yaml
templates/slapd/           # cn=config LDIFs (base config, mdb db, overlays)
templates/ldif/            # directory-tree LDIFs (People/Groups/Computers/Policies/...)
templates/ldapmodify/      # post-config ldapmodify LDIFs (ACLs, SSL authz)
templates/other/sudo.ldif  # sudoRole schema definition
ldif-files/                # reference/scratch LDIF snippets, not loaded by any playbook
vars.yaml                  # default variables (non-secret)
```

Files under `templates/*/` are auto-discovered (globbed and sorted by filename), so the numeric
prefixes (`01-`, `02-`, ...) control load order — keep that in mind when adding new overlays or
tree entries. A `.ldif.disabled` extension (see `templates/ldif/60-radius.ldif.disabled`) excludes
a file from the glob, which is how the (currently unused) RADIUS OU is kept out of the build.

## Security model

The production ACL set (`templates/ldapmodify/01-acls.ldif`) is the important one to read before
deploying:

- Anonymous binds can authenticate but not browse the directory.
- `nss`, `sudo`, and `radius` service accounts get only the read access their function needs.
- ppolicy state, shadow attributes, and identity attributes are hidden from regular users and
  only writable by admin or the local root SASL EXTERNAL identity.
- Users can update their own contact info but not structural/identity attributes.

## Known gaps

- `reset-password.yaml` is a bare task list (no `hosts:`/play wrapper) and isn't included by any
  playbook — it isn't directly runnable as-is; wrap it in a play (see `service-accounts.yaml` for
  the pattern) before use, or include it from `add-users.yaml`.
- `ldif-files/` looks like leftover/reference material — its `07-service_accounts.ldif` duplicates
  `templates/ldif/70-service_accounts.ldif` and isn't referenced by `config_template_dir`,
  `ldap_tree_template_dir`, or `post_config_ldapmodify_dir`.
