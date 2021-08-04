Atlassian JIRA Software
=========

[![Project Status: Unsupported – The project has reached a stable, usable state but the author(s) have ceased all work on it. A new maintainer may be desired.](https://www.repostatus.org/badges/latest/unsupported.svg)](https://www.repostatus.org/#unsupported)

Install and configure Atlassian JIRA Software from tarball. Front-end it with a nginx, listening with TLS 1.2 using a self-signed certificate.

Requirements
------------

Outbound Internet connectivity.

Role Variables
--------------

```
java_application: java-11-openjdk

client_max_body_size: 100M
jira_version: 8.4.1
jira_download: "https://product-downloads.atlassian.com/software/jira/downloads/atlassian-jira-software-{{ jira_version }}.tar.gz"

dhparams_bits: 3072
server_name: jira.example.com
ssl_certificate: /etc/ssl/certs/snakeoil.crt
ssl_certificate_key: /etc/ssl/certs/snakeoil.pem
ssl_dhparam: /etc/ssl/certs/dhparams.pem

ssl_size: 3072
ssl_type: RSA

ssl_country_name: AU
ssl_state_or_province_name: Some-State
ssl_locality_name: ""
ssl_organization_name: Internet Widgits Pty Ltd
ssl_organizational_unit_name: ""
```

Dependencies
------------

None.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: tag_function_jira:&tag_component_jira_software

      roles:
         - deekayen.jira_software
