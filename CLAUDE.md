# pxe-map-api-ansible

Ansible playbook that installs/configures/upgrades
[pxe-map-api](https://github.com/team-redbull/pxe-map-api) (a small FastAPI
service; see that repo's `SPEC.md` for full background) on 5 per-site PXE
VMs (RHEL 9.4, SELinux enforcing). This repo is deployment-only - there is
no application code here, and no application logic should be added here
either (e.g. don't add tasks that patch app behavior; fix the app repo and
let this playbook re-deploy it).

## Commands

```bash
ansible-galaxy collection install -r requirements.yml   # community.general, ansible.posix
ansible-playbook site.yml --syntax-check
ansible-lint site.yml                                     # should stay clean at the "production" profile
ansible-playbook site.yml --ask-vault-pass                 # all 5 sites
ansible-playbook site.yml --ask-vault-pass --limit pxe-site1
```

## Architecture

- `site.yml` - top-level playbook, applies the one role to the
  `pxe_map_api_sites` inventory group.
- `roles/pxe_map_api/` - the only role. `tasks/main.yml` imports, in order:
  `preflight.yml` (refuses to deploy an empty `PXE_MAP_TOKEN` unless
  `pxe_map_api_allow_no_auth: true` is explicit, and checks the app repo is
  where `pxe_map_api_src_dir` says it is), `user.yml` (system user +
  `PXE_MAP_DIR`), `selinux.yml` (`community.general.sefcontext` +
  `restorecon`, skipped when SELinux isn't enabled), `app.yml` (copies
  `app/` + `requirements.txt` from `pxe_map_api_src_dir`, builds the venv
  from the local pip mirror only - never PyPI), `service.yml` (renders
  `templates/env.j2` and `templates/pxe-map-api.service.j2`, enables +
  starts the service, restarts on any change via `handlers/main.yml`).
- `inventory/hosts.yml` - the 5 site VMs, group `pxe_map_api_sites`.
- `inventory/group_vars/pxe_map_api_sites.yml` - vars shared by all 5
  sites (pip mirror path, `PXE_MAP_DIR`).
- `inventory/host_vars/<site>/vars.yml` - non-secret per-site overrides
  (e.g. `pxe_map_api_bind`).
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
  `CLAUDE.md`.
- No new collections beyond `community.general` / `ansible.posix`
  (`requirements.yml`) without a strong reason - keep the control-node
  tooling footprint small, same air-gapped-friendly spirit as the app's
  "no new runtime deps" rule.
- Keep `ansible-lint site.yml` clean at the `production` profile - it was
  clean when this repo was created; a new violation means something
  regressed, not that the rule should be suppressed.
