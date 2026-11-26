# Nginx Static Site

Serves a directory tree as a browsable, read-only website using an unprivileged Nginx container.

## Usage

Set `nginx_static_site_enabled: true` and configure the site variables in `inventories/<your_inventory>/group_vars/nas.yml`. For example:

```yaml
nginx_static_site_enabled: true
nginx_static_site_root: /NAS/documents/manuals/jeep/2012_wrangler_service_information
nginx_static_site_index: common/html/default.htm
nginx_static_site_bind_address: "{{ ansible_facts['default_ipv4']['address'] }}"
nginx_static_site_port: "8098"
nginx_static_site_allowlist:
  - 192.168.2.0/24
```

The site is then available at `http://<nas-lan-ip>:8098/`. The root URL redirects to the configured index file so its relative links resolve correctly. No Traefik route or external-access option is configured.

## Permissions and access

The role does not change ownership or permissions in the document root. To align the tree once with the repository's shared-folder convention, run these commands on the NAS:

```bash
sudo chown -R ansible-nas:ansible-nas -- /NAS/documents/manuals/jeep/2012_wrangler_service_information/
sudo chmod -R u=rwX,g=rwX,o=rX -- /NAS/documents/manuals/jeep/2012_wrangler_service_information/
```

This sets directories to `0775` and non-executable files to `0664`, preserving existing execute bits. Other local users can read the content; the container mount itself is read-only. Nginx denies dotfiles, including `.git`, and does not follow symbolic links.

Nginx allows only source networks in `nginx_static_site_allowlist` and denies all others. The role binds the published port to the configured host address, not all interfaces. Network firewalls should also restrict the port to your LAN subnet; Docker-published ports can bypass UFW rules. Before starting, the role checks that the port is not already listening (except when its container already exists, to support idempotent runs). The image tag defaults to `stable-alpine`; pin a release tag or digest if reproducible deployments are required.

Change the inventory variables to serve another directory or run another instance. The source directory must exist before enabling the role.
