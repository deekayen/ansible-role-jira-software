# deekayen.jira_software

[![CI](https://github.com/deekayen/ansible-role-jira-software/actions/workflows/ci.yml/badge.svg)](https://github.com/deekayen/ansible-role-jira-software/actions/workflows/ci.yml) [![Ansible Galaxy](https://img.shields.io/badge/galaxy-deekayen.jira__software-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/deekayen/jira_software/) [![Project Status: Unsupported – The project has reached a stable, usable state but the author(s) have ceased all work on it. A new maintainer may be desired.](https://www.repostatus.org/badges/latest/unsupported.svg)](https://www.repostatus.org/#unsupported) ![BSD 3-Clause license](https://img.shields.io/badge/license-BSD%203--Clause-blue)

> **Deprecated.** Atlassian ended Jira Server in February 2024, and this role
> installs Jira Software 8.4.1 on EL 7, which is also end of life. It is kept
> for existing installs. CI lints and syntax-checks it but no longer
> converges it on a running system.

An Ansible role that installs Atlassian Jira Software from the standalone tarball on an EL 7 host and puts nginx in front of it on port 443 with a self-signed TLS certificate.

The role installs the `java_application` package, creates a `jira` user and group (uid and gid 50000), and unpacks `jira_download` into `/opt/atlassian/jira/atlassian-jira-software-<version>-standalone`, with `/opt/atlassian/jira/current` linked to it. It sets `jira.home` to `/var/atlassian/jira`, writes `/var/atlassian/jira/cluster.properties` and the Tomcat `conf/server.xml`, and installs the SysV init script from `files/jira` as `/etc/init.d/jira`. For nginx, it generates Diffie-Hellman parameters and a self-signed key, CSR, and certificate with `community.crypto`, then writes `/etc/nginx/conf.d/jira.conf`, which proxies to Tomcat on `127.0.0.1:8080`. nginx itself comes from the `nginxinc.nginx` role dependency.

The Galaxy name is `deekayen.jira_software`, with an underscore.

## Requirements

- ansible-core 2.15 or newer on the controller, and 2.16 or older to manage an EL 7 target. As of October 2026, the [Ansible support matrix](https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html) lists Python 2.7 and 3.6 as target versions for 2.16 but not 2.17, and EL 7 ships those two Pythons.
- The `ansible.posix`, `community.crypto`, and `community.general` collections, and the `nginxinc.nginx` and `geerlingguy.pip` roles. `tests/requirements.yml` lists all five.
- An EL 7 target with outbound HTTPS to `jira_download` and to the package repositories for Java, nginx, and pip.
- Privilege escalation for the whole play. Most tasks install packages or write under `/etc` and `/opt` without a task-level `become`; the tarball tasks switch to `become_user: jira`.
- Fact gathering left on. The certificate uses `ansible_facts.fqdn` and `ansible_facts.hostname`.

## Supported platforms

| Platform | Versions |
| --- | --- |
| EL | 7 |

CI runs `ansible-lint` and `ansible-playbook --syntax-check` only.

## Installation

From Ansible Galaxy:

```bash
ansible-galaxy role install deekayen.jira_software
ansible-galaxy collection install ansible.posix community.crypto community.general
```

Or pin it in `requirements.yml`:

```yaml
---
roles:
  - name: deekayen.jira_software
    src: https://github.com/deekayen/ansible-role-jira-software.git
    scm: git
    version: main

collections:
  - name: ansible.posix
  - name: community.crypto
  - name: community.general
```

```bash
ansible-galaxy install -r requirements.yml
```

Galaxy resolves `nginxinc.nginx` and `geerlingguy.pip` from `meta/main.yml` in both cases.

## Role variables

| Variable | Default | Description |
| --- | --- | --- |
| `java_application` | `java-11-openjdk` | Java package installed with the system package manager. |
| `client_max_body_size` | `100M` | nginx `client_max_body_size` for uploads. Must be digits with an optional `k`, `m`, or `g` suffix; the role asserts this. |
| `jira_version` | `8.4.1` | Jira Software version. Used in the default `jira_download` URL and in every path under `/opt/atlassian/jira`. |
| `jira_download` | `https://product-downloads.atlassian.com/software/jira/downloads/atlassian-jira-software-{{ jira_version }}.tar.gz` | Tarball URL. Must be an `http://` or `https://` URL ending in `.tar.gz`. Point it at an internal mirror if the host can't reach Atlassian. |
| `dhparams_bits` | `3072` | Size of the generated Diffie-Hellman parameters. Must be at least 2048. |
| `server_name` | `jira.example.com` | Placeholder. Set it to the Jira site name; it becomes the nginx `server_name` and the Tomcat connector `proxyName`. |
| `ssl_certificate` | `/etc/ssl/certs/snakeoil.crt` | Certificate path in the nginx configuration. See [Known issues](#known-issues). |
| `ssl_certificate_key` | `/etc/ssl/certs/snakeoil.pem` | Private key path in the nginx configuration. See [Known issues](#known-issues). |
| `ssl_dhparam` | `/etc/ssl/certs/dhparams.pem` | Where the role writes the Diffie-Hellman parameters, and the path nginx reads them from. |
| `ssl_size` | `3072` | Size of the generated private key. Must be at least 2048. |
| `ssl_type` | `RSA` | Type of the generated private key. Must be one of `RSA`, `DSA`, `ECC`, `Ed25519`, `Ed448`, `X25519`, or `X448`. |
| `ssl_country_name` | `AU` | Country in the self-signed certificate subject. |
| `ssl_state_or_province_name` | `Some-State` | State or province in the certificate subject. |
| `ssl_locality_name` | `""` | Locality in the certificate subject. |
| `ssl_organization_name` | `Internet Widgits Pty Ltd` | Organization in the certificate subject. |
| `ssl_organizational_unit_name` | `""` | Organizational unit in the certificate subject. |

## Behavior

- The certificate's common name is `ansible_facts.fqdn`, and its subject alternative names are `inventory_hostname` and `ansible_facts.hostname`. `server_name` is not in the certificate.
- nginx listens on 443 for IPv4 and IPv6 with HTTP/2, allows TLS 1.2 only, and sends `Strict-Transport-Security: max-age=63072000`.
- The SELinux `httpd_t` domain is set to permissive so nginx can connect to Tomcat. The comment in `tasks/main.yml` explains why the narrower `httpd_can_network_connect` boolean was not used.
- firewalld gets a permanent `https` rule only if `/usr/bin/firewall-cmd` exists.
- Changing `jira_version` unpacks a new tree beside the old one, repoints `current`, and restarts Jira. The old tree stays in `/opt/atlassian/jira`.
- `cluster.properties` sets `jira.node.id` to `inventory_hostname` and `jira.shared.home` to `/var/atlassian/share`.
- `geerlingguy.pip` installs `PyOpenSSL>=16.2.0` and `cryptography>=1.6` on the target before this role runs.

## Dependencies

- [nginxinc.nginx](https://galaxy.ansible.com/ui/standalone/roles/nginxinc/nginx/), with its own defaults.
- [geerlingguy.pip](https://galaxy.ansible.com/ui/standalone/roles/geerlingguy/pip/), with `pip_install_packages` set to `PyOpenSSL>=16.2.0` and `cryptography>=1.6`.

## Example playbook

```yaml
---
- name: Install Jira Software behind nginx.
  hosts: jira_el7
  become: true

  vars:
    server_name: jira.example.internal
    jira_download: https://mirror.example.internal/atlassian/atlassian-jira-software-8.4.1.tar.gz
    ssl_organization_name: Example Engineering

  roles:
    - deekayen.jira_software
```

`jira.example.internal` and `mirror.example.internal` are placeholders.

## Known issues

- `tasks/main.yml` lines 116, 130, 147, 209, and 218 pass `ansible.builtin.group:` as a parameter to the `user`, `file`, `copy`, and `template` modules instead of `group:`. Each of those modules rejects it as an unsupported parameter, so no run gets past "Create local jira user."
- "Setup nginx cert directory." (`tasks/main.yml` line 13) uses `state: link` on `/etc/ssl/certs` with no `src`. It fails when that path is a directory and changes nothing when it is already a symlink.
- The private key, CSR, and certificate are always written to `/etc/ssl/certs/snakeoil.pem`, `snakeoil.csr`, and `snakeoil.crt`. `ssl_certificate` and `ssl_certificate_key` only change the paths in the nginx configuration, so pointing them at a real certificate still leaves the role creating the self-signed files at the fixed paths.
- `files/jira` exports `JAVA_HOME=/usr/java/default`. No task creates that path, and it does not follow `java_application`.

## Development

CI runs on every push to `main` and every pull request (see `.github/workflows/ci.yml`). It installs `tests/requirements.yml`, runs `ansible-lint --profile production`, and syntax-checks `tests/test.yml`. To run the same checks locally:

```bash
pip3 install ansible-lint
ansible-galaxy install -r tests/requirements.yml
ansible-lint --profile production
mkdir -p .ansible/roles && ln -sfn "$PWD" .ansible/roles/deekayen.jira_software
ANSIBLE_ROLES_PATH=.ansible/roles:~/.ansible/roles ansible-playbook --syntax-check tests/test.yml -i tests/inventory
```

The repository also has a `.pre-commit-config.yaml`; run `pre-commit run --all-files` before pushing.

### Repository layout

| Path | Purpose |
| --- | --- |
| `tasks/main.yml` | TLS material, firewalld, SELinux, Java, the `jira` account, the tarball, and configuration files. |
| `tasks/assert.yml` | Input validation, tagged `always`. |
| `handlers/main.yml` | `chkconfig --add jira`, Jira start and restart, nginx reload, and firewalld reload. |
| `templates/` | `server.xml.j2`, `cluster.properties.j2`, and `nginx.conf.j2`. |
| `files/jira` | SysV init script installed as `/etc/init.d/jira`. |
| `defaults/main.yml` | Every user-facing variable. |
| `tests/` | Syntax-check playbook, inventory, and test requirements used by CI. |

## Releases

Pushing a git tag runs `.github/workflows/release.yml`, which imports the tagged commit into Ansible Galaxy as `deekayen.jira_software`. The import needs a `GALAXY_API_KEY` repository or organization secret.

## License

BSD 3-Clause. See [LICENSE](LICENSE).

## Author

[David Norman](https://github.com/deekayen). Sponsorship links are in [.github/FUNDING.yml](.github/FUNDING.yml).
