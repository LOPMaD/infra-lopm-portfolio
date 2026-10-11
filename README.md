# Infrastructure (lopm.space)
<img width="1103" height="912" alt="image" src="https://github.com/user-attachments/assets/09fcb862-f3cd-4949-8b71-42826fb0c107" />

**Live:** [lopm.space](https://lopm.space)

**Stack:** VPS, Ansible, UFW, Nginx, Certbot, Docker, GitLab Runner

## Overview

Infrastructure repository for my portfolio site. It's a set of Ansible roles that provisions a fresh server with everything needed to run the project in two commands.

The site's static files and CI/CD setup live in a separate repository, [lopm-portfolio](https://github.com/LOPMaD/lopm-portfolio). This is intentional: splitting infrastructure and application code across two repositories is common practice, so developers work in the application repo while the infrastructure lives on its own.

## Architecture

```
Internet
   │
   ▼
UFW (only ports 22, 80, 443 allowed)
   │
   ▼
Nginx (reverse proxy + TLS)
   │
   ▼
Docker containers (the site and other services)
```

- **UFW**: firewall, allows traffic only on ports 22, 80 and 443.
- **Nginx**: reverse proxy, forwards user requests to the Docker containers.
- **Docker**: containerization; all services, including the site itself, come up with a single command.
- **Certbot**: issues TLS certificates for the domains and secures data transfer.
- **GitLab Runner**: waits for pushes to the application repo and deploys updates automatically (CI/CD details are in the [application repository](https://github.com/LOPMaD/lopm-portfolio)).

## Prerequisites

Before running the playbooks:

- A registered domain (in my case `lopm.space`) with DNS A-records for the root domain and every subdomain, all pointing to the server's IP.
- A VPS running a clean Ubuntu install with SSH access.
- An SSH key for the server.
- A GitLab account with Container Registry and CI/CD variables configured.
- Ansible and Git installed locally.

## Project Structure

```
.
├── ansible.cfg
├── bootstrap.yaml            # one-time server initialization
├── site.yaml                 # main playbook (roles)
├── .pre-commit-config.yaml   # gitleaks as a pre-commit hook
├── inventory/
│   └── production.yaml       # target server
└── roles/
    ├── ufw/                  # firewall
    ├── nginx/                # Nginx install + base config
    ├── certbot/              # TLS certificates
    ├── nginx_site/           # working Nginx config with TLS
    ├── docker/               # Docker install
    └── gitlab-runner/        # CI/CD runner
```

## Bootstrap

There are two playbooks:

- **`bootstrap.yaml`**: run once on a fresh server for the initial setup.
- **`site.yaml`**: the main playbook, split into roles.

What `bootstrap.yaml` does:

- Creates the **ansible** user. Ansible runs as root by default, but for security, root SSH login is disabled. Ansible is agentless (it works over SSH and Python), so the ansible user receives the public SSH key that the playbooks are deployed with.
- Creates the **vpsadmin** user, an admin account for SSH access to the server.
- Disables root SSH login (edits `sshd_config`).
- Restarts the SSH service to apply the new config.

## Roles

`site.yaml` is split into the following roles:

**ufw**: firewall. Default-deny policy: anything not explicitly allowed is blocked.

**nginx**: installs Nginx. Adds the official Nginx repository (distro repositories usually ship an outdated version). Includes a check for an existing certificate: if there isn't one yet, a base config without TLS is deployed. This is needed so certbot can pass its config validation when issuing the first certificate (see Troubleshooting).

**certbot**: installs certbot and its dependencies, then issues the certificate. The email and domain list are passed as Ansible variables, and the domain array is expanded into the command via a template. A check for an existing certificate prevents re-issuing. Two cron jobs are added: certificate renewal and certbot self-update.

**nginx_site**: deploys the working Nginx config with all certificates and settings: proxy paths, domains, HTTP-to-HTTPS redirect, and blocking of requests to non-existent subdomains and paths.

**docker**: installs Docker. Works together with GitLab Runner (docker-compose and CI details are in the [application repository](https://github.com/LOPMaD/lopm-portfolio)).

**gitlab-runner**: installs GitLab Runner, creates a dedicated user for the runner and adds it to the docker group. Checks whether the runner is registered and registers it if not. The registration token is stored in Ansible Vault (encrypted) and decrypted when the playbook runs.

## Security

- **Root SSH login disabled**: reduces the risk of gaining root access over SSH.
- **Password SSH login disabled**: removes the risk of brute-forcing user passwords.
- **A dedicated user per process** instead of a shared root account: separates privileges and makes logs clearer (you can see which user did what).
- **Gitleaks** as a pre-commit hook: scans commits and blocks them if a secret is found, warning about a potential token leak.
- **Secrets in Ansible Vault**: tokens and passwords are stored encrypted, not in plain text in the repository.

## Troubleshooting

**Nginx + Certbot.** At first I wrote the Nginx config with the certificate paths and SSL already enabled. The playbook failed: when issuing a certificate, certbot validates the Nginx config, saw the SSL lines pointing to key files that didn't exist yet, and validation failed. The fix was to split Nginx into two roles: `nginx` installs the server and, if no certificate exists, deploys a base config (just to pass validation), while `nginx_site` deploys the full config with TLS after the certificates are issued.

**ERR_CONNECTION_CLOSED.** The browser returned this error when opening the domain. The cause: the `ssl` mode wasn't enabled on port 443 in Nginx. The browser tried to establish a secure TLS connection on 443, but the server responded with plain HTTP, so the connection was dropped.

**GitLab Runner (shell executor).** A job failed with `prepare environment: exit status 1`. The cause was the `.bash_logout` file in the runner user's home directory: on job completion it calls `clear_console`, which fails without a terminal. The fix was to remove `.bash_logout`.
