# blog-bind-recursive-resolver

Companion code for the blog post
[How to Setup BIND as a Recursive DNS Resolver](https://blog.bsdserver.nl/how-to-setup-bind-as-a-recursive-dns-resolver/).

It builds the post's setup: a caching recursive BIND 9 resolver that only answers the
networks in the `trusted` ACL, validates DNSSEC, hides its version, rate limits responses,
refuses to hand out loopback addresses (`deny-answer-addresses`) and logs to
`/var/log/named/` (default, query and security channels). It downloads current root hints
from InterNIC and points the existing hint zone at them, as in section 6. Forwarding
(section 4) is optional and off by default.

## What is in here

| Path | Purpose |
|------|---------|
| `terraform/` | One Ubuntu 24.04 VM on Proxmox VE (bpg/proxmox provider, cloud-init), outputs an Ansible inventory |
| `ansible/site.yml` | One playbook for the whole setup |
| `ansible/roles/bind` | Installs BIND and writes `named.conf` with includes for options and logging |
| `ansible/roles/bind_resolver` | ACL, options, hardening, optional forwarders, logging and root hints from the post |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VM)
- ansible-core >= 2.15 (tested with 2.20.1)
- Target host: Ubuntu 24.04 (tested, BIND 9.18) or RHEL/Rocky 9 (handled, not lab-tested)
- Outbound DNS (UDP/TCP 53) from the resolver to the internet

## Quick start

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init && terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

cd ../ansible                                   # or: cp inventory.example.yml inventory.yml
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

# Section 8 of the post
dig @10.0.0.53 google.com
dig @10.0.0.53 dnssec-failed.org                # SERVFAIL: DNSSEC validation failure
dig @10.0.0.53 +short version.bind txt chaos    # "not disclosed"
```

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in `roles/bind_resolver/defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `bind_resolver_trusted` | localhost, localnets, RFC 1918 ranges | Contents of `acl "trusted"` |
| `bind_resolver_forwarders` | `[]` | Upstream resolvers; empty means full recursion |
| `bind_resolver_forward` | `first` | `first` (fall back to recursion) or `only` |
| `bind_resolver_max_cache_size` | `256m` | `max-cache-size` |
| `bind_resolver_max_cache_ttl` / `bind_resolver_max_ncache_ttl` | `86400` / `3600` | Cache TTL caps |
| `bind_resolver_rrl_responses_per_second` / `bind_resolver_rrl_window` | `10` / `5` | Response rate limiting |
| `bind_resolver_querylog` | `false` | `querylog` at startup |
| `bind_resolver_update_root_hints` | `true` | Download `named.root` and use it for the hint zone |
| `bind_resolver_log_dir` | `/var/log/named` | Log directory |

This setup has no secrets, so `group_vars/all/vault.yml.example` is empty.

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container (BIND 9.18.39) with the
post's `dig` checks afterwards, plus a query from an address outside the ACL (REFUSED,
logged in `security.log`); the second run reports `changed=0`. ansible-lint (production
profile), yamllint, `terraform validate` and tflint are clean. The Terraform code has not
been applied against a real Proxmox cluster.

## License

MIT
