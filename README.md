# pxe-map-api-ansible

Ansible automation that installs and configures
[pxe-map-api](https://github.com/team-redbull/pxe-map-api) on the 5 per-site
PXE VMs (RHEL 9.4, SELinux enforcing). This repo is deployment-only - no
application code lives here; see `pxe-map-api` for that (and for the *why*
behind the app's design, `SPEC.md` there).

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
#    default), and set pxe_map_api_local_mirror in
#    inventory/group_vars/pxe_map_api_sites.yml

# 4. Per site, generate + vault-encrypt a token (see DEPLOY.md step 3):
cd inventory/host_vars/pxe-site1
cp vault.yml.example vault.yml
# edit vault.yml with a real `openssl rand -hex 32` token
ansible-vault encrypt vault.yml
cd -

# 5. Run it
ansible-playbook site.yml --ask-vault-pass
```

Idempotent, targets all 5 sites in parallel by default
(`--limit pxe-site1` for just one).

## Structure

```
site.yml                                    top-level playbook
roles/pxe_map_api/                          the one role, does everything
inventory/hosts.yml                         the 5 site VMs
inventory/group_vars/pxe_map_api_sites.yml  shared vars (pip mirror, dir)
inventory/host_vars/<site>/vars.yml         per-site overrides (e.g. bind)
inventory/host_vars/<site>/vault.yml        per-site PXE_MAP_TOKEN (vault-encrypted, one per site)
```
