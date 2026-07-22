# pxe-map-api-ansible

Ansible playbook that provisions the 5 per-site PXE VMs (RHEL 9.4, SELinux
enforcing) end to end, in two layers:

1. `pxe_server` - the TFTP (dnsmasq) + nginx boot-chain prerequisites (iPXE
   firmware over TFTP, `/var/www/pxe-map/` served read-only over HTTP).
2. `pxe_map_api` - installs/configures/upgrades
   [pxe-map-api](https://github.com/team-redbull/pxe-map-api) (a small
   FastAPI service; see that repo's `SPEC.md` for full background), which
   owns writing the per-MAC files that `pxe_server`'s nginx serves.

This repo is deployment-only - there is no application code here, and no
application logic should be added here either (e.g. don't add tasks that
patch app behavior; fix the app repo and let this playbook re-deploy it).

## Commands

```bash
ansible-galaxy collection install -r requirements.yml   # community.general, ansible.posix
ansible-playbook site.yml --syntax-check
ansible-lint site.yml                                     # should stay clean at the "production" profile

ansible-playbook site.yml --ask-vault-pass                          # both roles, all 5 sites
ansible-playbook site.yml --ask-vault-pass --tags pxe_server         # layer 1 only (nginx/dnsmasq), all sites
ansible-playbook site.yml --ask-vault-pass --tags pxe_map_api        # layer 2 only (the API app), all sites
ansible-playbook site.yml --ask-vault-pass --limit pxe-site1         # both roles, one site
ansible-playbook site.yml --ask-vault-pass --tags pxe_server --limit pxe-site1   # layer 1, one site
```

## Architecture

- `site.yml` - top-level playbook, applies both roles (in order) to the
  `pxe_map_api_sites` inventory group. `pxe_server` is tagged
  `pxe_server`, `pxe_map_api` is tagged `pxe_map_api`, so either can be run
  alone with `--tags`; with no `--tags` both run, in that order.
- `roles/pxe_server/` - layer 1. Single `tasks/main.yml`: installs
  `dnsmasq`/`nginx`/`tftp-server`/`ipxe-bootimgs-x86`/
  `policycoreutils-python-utils`, copies `ipxe.efi` into the TFTP root,
  templates `boot.ipxe` (iPXE variables like `${net0/mac:hexraw}` are
  wrapped in `{% raw %}` - see the comment in
  `templates/boot.ipxe.j2`, this is the easiest thing to break here),
  templates `/etc/dnsmasq.conf` (TFTP-only, no DNS/DHCP), disables the stock
  nginx default server block's `listen 80` directives (idempotent regex,
  not a full block delete), templates `/etc/nginx/conf.d/pxe-map.conf`,
  creates the PXE map directory (owner `nginx`), sets SELinux fcontexts
  (`httpd_sys_content_t` on the PXE map dir, `tftpdir_rw_t` on the TFTP
  root, both skipped when SELinux isn't enabled), opens firewalld
  (TFTP + HTTP services, the dnsmasq TFTP data port range), and
  enables/starts both services. Handlers restart on config change only.
  See `roles/pxe_server/README.md` for the full variable list and the boot
  chain it implements.
- `roles/pxe_map_api/` - layer 2. `tasks/main.yml` imports, in order:
  `preflight.yml` (refuses to deploy an empty `PXE_MAP_TOKEN` unless
  `pxe_map_api_allow_no_auth: true` is explicit, and checks the app repo is
  where `pxe_map_api_src_dir` says it is), `user.yml` (system user +
  `PXE_MAP_DIR`), `selinux.yml` (`community.general.sefcontext` +
  `restorecon`, skipped when SELinux isn't enabled), `app.yml` (copies
  `app/` + `requirements.txt` from `pxe_map_api_src_dir`, builds the venv
  from the local pip mirror only - never PyPI), `service.yml` (renders
  `templates/env.j2` and `templates/pxe-map-api.service.j2`, enables +
  starts the service, restarts on any change via `handlers/main.yml`).
- `inventory/hosts.yml` - the 5 site VMs, group `pxe_map_api_sites` (kept as
  the group name even though it now provisions both layers, to avoid an
  unnecessary inventory rename).
- `inventory/group_vars/pxe_map_api_sites.yml` - vars shared by all 5
  sites (pip mirror path, `PXE_MAP_DIR`).
- `inventory/host_vars/<site>/vars.yml` - non-secret per-site overrides:
  `pxe_map_api_bind`, and the `pxe_server` role's required
  `pxe_server_vm_ip` / `pxe_server_interface` (no sensible shared default -
  every site's VM has a different provisioning IP/NIC).
- `inventory/host_vars/<site>/vault.yml` - that site's `PXE_MAP_TOKEN`,
  `ansible-vault`-encrypted. **One token per site, never shared** - if one
  site's PXE VM is compromised, the others must not be. `vault.yml.example`
  is the committed plaintext template; `vault.yml` itself is safe to commit
  once encrypted, never before.

## Conventions / constraints

- `pxe_map_api_src_dir` (role default: `{{ playbook_dir }}/../pxe-map-api`,
  i.e. this repo and the app repo checked out as siblings) is how this
  playbook finds the app code to copy - this repo has none of its own.
  Don't vendor a copy of `app/` in here; that defeats the point of having
  two repos.
- The preflight no-token check exists on purpose (mirrors
  `app/config.py`'s own runtime warning in the app repo, enforced earlier
  and harder here because the blast radius is 5 VMs, not one dev's
  laptop). Don't remove or weaken it without checking with the user - it
  was an explicit, confirmed design decision, not a default that crept in.
- Never add a task that disables SELinux (`setenforce 0`) or loosens
  `/var/www/pxe-map` permissions - same hard rule as the app repo's
  `CLAUDE.md`. This applies to both roles.
- `pxe_server` and `pxe_map_api` both manage `/var/www/pxe-map` (one as
  `nginx`-owned, so it exists before the API is installed; the other as
  `pxe-map-api`-owned once the app role runs). `site.yml` always runs
  `pxe_server` first so a full run converges to a stable owner - if you
  alternate `--tags pxe_server` / `--tags pxe_map_api` runs on the same
  host, expect the owner to flip each time (harmless, mode `0755` keeps it
  world-readable regardless). Don't try to eliminate this by having either
  role skip the directory - see `roles/pxe_server/README.md`.
- No new collections beyond `community.general` / `ansible.posix`
  (`requirements.yml`) without a strong reason - keep the control-node
  tooling footprint small, same air-gapped-friendly spirit as the app's
  "no new runtime deps" rule. `pxe_server`'s firewalld tasks use
  `ansible.posix.firewalld` - no new collection needed.
- Keep `ansible-lint site.yml` clean at the `production` profile - it was
  clean when this repo was created; a new violation means something
  regressed, not that the rule should be suppressed.
