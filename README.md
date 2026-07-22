# pxe-map-api-ansible

Ansible automation that provisions the 5 per-site PXE VMs (RHEL 9.4,
SELinux enforcing) end to end, in two layers:

1. **`pxe_server`** - TFTP (dnsmasq) + nginx boot-chain prerequisites (iPXE
   firmware over TFTP, `/var/www/pxe-map/` served read-only over HTTP).
2. **`pxe_map_api`** - installs/configures/upgrades
   [pxe-map-api](https://github.com/team-redbull/pxe-map-api) itself, the
   FastAPI service that manages the per-MAC files layer 1's nginx serves.

This repo is deployment-only - no application code lives here; see
`pxe-map-api` for that (and for the *why* behind the app's design,
`SPEC.md` there).

See `DEPLOY.md` for the full step-by-step guide. See `CLAUDE.md` for the
role/inventory structure if you're changing this automation.

## Quickstart

```bash
# 1. Have both repos checked out as siblings:
#    <parent>/pxe-map-api
#    <parent>/pxe-map-api-ansible   (this repo)

# 2. Install the required collections
ansible-galaxy collection install -r requirements.yml

# 3. Point inventory/hosts.yml at your 5 real site VMs (placeholders by
#    default). Set pxe_map_api_local_mirror in
#    inventory/group_vars/pxe_map_api_sites.yml, and each site's
#    pxe_server_vm_ip / pxe_server_interface in
#    inventory/host_vars/<site>/vars.yml (placeholders by default).

# 4. Per site, generate + vault-encrypt a token (see DEPLOY.md step 3):
cd inventory/host_vars/pxe-site1
cp vault.yml.example vault.yml
# edit vault.yml with a real `openssl rand -hex 32` token
ansible-vault encrypt vault.yml
cd -

# 5. Run it - both layers, all 5 sites:
ansible-playbook site.yml --ask-vault-pass
```

Idempotent, targets all 5 sites in parallel by default.

## Running just one layer, or one site

Both roles are tag-selectable, and the inventory group works with the
usual `--limit`:

```bash
# Just the nginx/dnsmasq boot-chain prerequisites (no API app):
ansible-playbook site.yml --ask-vault-pass --tags pxe_server

# Just the pxe-map-api app (assumes pxe_server has already been applied once):
ansible-playbook site.yml --ask-vault-pass --tags pxe_map_api

# Either of the above, restricted to one site:
ansible-playbook site.yml --ask-vault-pass --tags pxe_server --limit pxe-site1

# Everything, one site only:
ansible-playbook site.yml --ask-vault-pass --limit pxe-site1
```

## Structure

```
site.yml                                    top-level playbook, both roles tagged for selective runs
roles/pxe_server/                           layer 1: TFTP (dnsmasq) + nginx boot-chain prerequisites
roles/pxe_map_api/                          layer 2: the pxe-map-api app itself
inventory/hosts.yml                         the 5 site VMs
inventory/group_vars/pxe_map_api_sites.yml  shared vars (pip mirror, dir)
inventory/host_vars/<site>/vars.yml         per-site overrides (bind, pxe_server_vm_ip, pxe_server_interface)
inventory/host_vars/<site>/vault.yml        per-site PXE_MAP_TOKEN (vault-encrypted, one per site)
```
