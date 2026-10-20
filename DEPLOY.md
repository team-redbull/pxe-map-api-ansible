# Deploying the PXE boot chain + pxe-map-api to the site PXE VMs

Target: 5 site PXE VMs (RHEL 9.4, SELinux enforcing), one per site. This
repo provisions **both layers** of each VM from scratch:

1. **`pxe_server`** - TFTP (dnsmasq) + nginx boot-chain prerequisites: iPXE
   firmware served over TFTP, `/var/www/pxe-map/` served read-only over
   HTTP. See `roles/pxe_server/README.md` for the full boot chain this
   implements.
2. **`pxe_map_api`** - installs/configures/upgrades
   [pxe-map-api](https://github.com/team-redbull/pxe-map-api) itself (see
   that repo's `SPEC.md` for full background), the service that writes the
   per-MAC files layer 1's nginx serves.

Both roles are tag-selectable, so you can run just layer 1, just layer 2,
or both (the default - see "Running just one layer" below). A brand-new
site VM needs layer 1 applied at least once before layer 2's files are
reachable over HTTP; after that, re-running either independently is fine.

## 0. Prerequisites

- A control node with SSH access to all 5 site VMs and `sudo`/root on each.
- `ansible-core` >= 2.15 on the control node:
  `python3 -m pip install --user ansible-core` (or your distro package).
- The required collections:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```
- Every site VM's `dnf`/yum already pointed at your local OS package
  mirror/Satellite (this repo doesn't configure repo files itself, just
  installs from whatever's configured) - it needs to provide `dnsmasq`,
  `nginx`, `tftp-server`, `ipxe-bootimgs-x86`, and
  `policycoreutils-python-utils` for the `pxe_server` role, and
  **`python3.12`** (RHEL 9.4+ AppStream) for the `pxe_map_api` role. Air-gapped:
  no internet access is assumed or used.
- A local **pip** package mirror reachable from every site VM (separate
  from the OS package mirror above), containing `fastapi==0.115.6`,
  `uvicorn==0.34.0`, `pydantic==2.10.4`, `python-dotenv==1.2.2` and their
  transitive deps (see the app repo's `requirements.txt` - always check
  that file, this list can drift out of date), **as wheels for Python 3.12**
  (`cp312` - `pydantic-core` is the compiled one). The `pxe_map_api` role runs
  `pip install --no-index --find-links=<mirror>` - it never touches PyPI.
  It builds the venv with `python3.12`, not the platform `python3`: RHEL 9's
  is 3.9, which the app cannot run on (it needs >= 3.10, and
  `python-dotenv==1.2.2` has no 3.9 release at all - verified on a UBI 9.4
  image). `pxe_map_api_python_version` overrides it.
- **A checkout of the `pxe-map-api` app repo as a sibling directory of this
  one**:
  ```
  <parent>/pxe-map-api            git@github.com:team-redbull/pxe-map-api.git
  <parent>/pxe-map-api-ansible    this repo
  ```
  This repo has no application code of its own - the role's default
  `pxe_map_api_src_dir` (`../pxe-map-api`) copies `app/` and
  `requirements.txt` from there. Override `pxe_map_api_src_dir` (in
  `inventory/group_vars/pxe_map_api_sites.yml`, or with `-e`) if you keep
  the app checkout somewhere else.

## 1. Point the inventory at your real VMs

Edit `inventory/hosts.yml` and replace the 5 placeholder `ansible_host`
values (`pxe-site1.example.com` ... `pxe-site5.example.com`) with your real
per-site PXE VM hostnames or IPs. The `pxe-site1` ... `pxe-site5` names on
the left are just inventory aliases - rename them to match your sites if
that's clearer (and rename the matching `inventory/host_vars/<name>/`
directory to match).

## 2. Set the local pip mirror

Edit `inventory/group_vars/pxe_map_api_sites.yml`:

```yaml
pxe_map_api_local_mirror: /path/to/your/local/pip-mirror
```

Override per-site in `inventory/host_vars/<site>/vars.yml` if a site's
mirror path differs.

## 3. Set each site's provisioning IP and NIC (for `pxe_server`)

Edit `inventory/host_vars/<site>/vars.yml` for each of the 5 sites and
replace the placeholder values:

```yaml
pxe_server_vm_ip: "10.10.1.5"    # this site's provisioning-facing IP -
                                  # templated into boot.ipxe's chain URL
pxe_server_interface: eth0        # NIC dnsmasq binds to on this VM
```

There's no sensible shared default across sites, so both are required
before running `--tags pxe_server` for real (the role defaults to
`CHANGE_ME`, which will template a broken `boot.ipxe` rather than fail the
run outright - fix it before relying on it).

## 4. Generate and vault a token per site

**Do not reuse one token across all 5 sites** - if one site's PXE VM is
compromised, the others must not be. For each site:

```bash
cd inventory/host_vars/pxe-site1   # repeat for site2..site5
cp vault.yml.example vault.yml
openssl rand -hex 32   # paste the output as pxe_map_api_token in vault.yml
ansible-vault encrypt vault.yml
```

You'll be prompted for a vault password (same or different per site, your
choice) - keep it somewhere safe; you need it on every future playbook run.
`vault.yml` (encrypted) is safe to commit; never commit the plaintext.

If you deliberately want a site to run unauthenticated (not recommended -
see the app repo's `CLAUDE.md`), skip the token and set
`pxe_map_api_allow_no_auth: true` in that site's `vars.yml` instead. The
playbook's preflight check refuses to deploy an empty token otherwise.

## 5. Set who may reach the API, and review the bind address per site

**Required:** the API port (the port of `pxe_map_api_bind`, `8080` by
default) is closed by firewalld until this role opens it, and the role opens
it only to the sources you list per site in
`inventory/host_vars/<site>/vars.yml` - the IPs/CIDRs that site's MCE
cluster calls the API from (its nodes' or egress IPs):

```yaml
pxe_map_api_allowed_sources:
  - "10.10.1.0/26"
```

The preflight check refuses to deploy a site with an empty list, the same
way it refuses an empty token - with nothing listed the MCE cannot reach the
API, and every IPMI install retries against it forever. To open the port to
every source instead (booting hosts included - not recommended), set
`pxe_map_api_firewall_open_to_all: true`. Rules are only added, never
removed: a source dropped from the list keeps its rule until you remove it
with `firewall-cmd --permanent --remove-rich-rule='...' && firewall-cmd --reload`.

Bind: `pxe_map_api_bind: "0.0.0.0:8080"` by default
(`roles/pxe_map_api/defaults/main.yml`) - listens on every interface, the
firewall rule above being what limits who gets in. For defense in depth,
override per site:

```yaml
pxe_map_api_bind: "10.10.1.5:8080"   # this site's management-facing IP
```

Never bind on the interface that serves PXE/TFTP/nginx traffic to booting
hosts.

## 6. Run it

```bash
ansible-playbook site.yml --ask-vault-pass
```

(`--vault-password-file <path>` instead of `--ask-vault-pass` if you keep
the vault password in a file - keep that file out of git.)

This targets all 5 sites in parallel, running **both** roles, by default.
To roll out to one site first: `ansible-playbook site.yml --ask-vault-pass
--limit pxe-site1`.

### Running just one layer

Both roles are tagged (`pxe_server`, `pxe_map_api`) so either can be run on
its own, on all sites or `--limit`ed to one:

```bash
# Just the nginx/dnsmasq boot-chain prerequisites, no API app:
ansible-playbook site.yml --ask-vault-pass --tags pxe_server

# Just the pxe-map-api app (layer 1 must already exist on that host):
ansible-playbook site.yml --ask-vault-pass --tags pxe_map_api

# Either, restricted to one site:
ansible-playbook site.yml --ask-vault-pass --tags pxe_server --limit pxe-site1
```

`site.yml` always applies `pxe_server` before `pxe_map_api` in a
default (untagged) run, so the ownership of `/var/www/pxe-map` always
converges the same way. If you only ever run one tag on a given host,
that's stable too. Alternating tags on the same host will make that
directory's owner flip between `nginx` and `pxe-map-api` each time -
harmless (mode `0755` keeps it world-readable either way) but you may see
it reported as a "changed" task. See `roles/pxe_server/README.md` for
more detail.

### What each role does, on each VM

`pxe_server` (layer 1):

1. Installs `dnsmasq`, `nginx`, `tftp-server`, `ipxe-bootimgs-x86`,
   `policycoreutils-python-utils` from the configured local mirror.
2. Copies `ipxe.efi` into `/var/lib/tftpboot/`.
3. Templates `/var/lib/tftpboot/boot.ipxe` (this site's `pxe_server_vm_ip`
   is the only templated value - the iPXE runtime variables in it are not
   Jinja-expanded).
4. Templates `/etc/dnsmasq.conf` in TFTP-only mode (no DNS, no DHCP).
5. Disables the stock nginx default server block's `listen 80` directives
   (idempotent - it no-ops once already disabled) so it doesn't conflict
   with the PXE map server block, and templates
   `/etc/nginx/conf.d/pxe-map.conf`.
6. Creates `/var/www/pxe-map` (owner `nginx`, mode `0755`) if it doesn't
   already exist as something else.
7. Registers SELinux fcontexts (`httpd_sys_content_t` on the PXE map dir,
   `tftpdir_rw_t` on the TFTP root) and runs `restorecon -Rv` on both.
   Skipped automatically on a non-SELinux host (e.g. testing in a
   container).
8. Opens firewalld for the `tftp` and `http` services and the dnsmasq TFTP
   data port range (`50000-50100/udp` by default), permanent + immediate.
9. Enables and starts `dnsmasq` and `nginx`; handlers restart either only
   if its config actually changed.

`pxe_map_api` (layer 2):

1. Fails the run (preflight) if that site has no token and hasn't
   explicitly opted into `pxe_map_api_allow_no_auth`, if it lists no
   `pxe_map_api_allowed_sources` and hasn't opted into
   `pxe_map_api_firewall_open_to_all`, or if the app repo isn't found at
   `pxe_map_api_src_dir` on the control node.
2. Creates the `pxe-map-api` system user (no login shell, no home dir).
3. Creates `/var/www/pxe-map` (dir `0755`, existing files re-chmod'd to
   `0644`), owned by that user.
4. Registers the persistent SELinux fcontext rule
   (`httpd_sys_content_t` on `/var/www/pxe-map(/.*)?`) and runs
   `restorecon -Rv` so the label survives reboots and reruns. Skipped
   automatically on a non-SELinux host.
5. Installs `python3.12`, copies `app/` and `requirements.txt` (from the
   sibling `pxe-map-api` checkout) into `/opt/pxe-map-api`, creates a venv
   there with `python3.12` - rebuilding one an earlier run made with another
   Python - and installs deps from the local pip mirror with
   `pip install --no-index` (never touches the network).
6. Renders `/etc/pxe-map-api/env` from that site's `vars.yml`/`vault.yml`
   (`PXE_MAP_DIR`, `PXE_MAP_BIND`, `PXE_MAP_TOKEN`) and the systemd unit,
   both from Jinja2 templates in `roles/pxe_map_api/templates/`.
7. Restarts the service if the env file, unit file, or app code changed;
   enables and starts it either way.
8. Opens the API port in firewalld to `pxe_map_api_allowed_sources` only
   (one rich rule per source), or to everyone with
   `pxe_map_api_firewall_open_to_all: true`.

Both roles are idempotent - rerun any time (e.g. after an app code change,
to pick up a rotated token, or to reapply layer 1's config) and only
what actually changed gets touched.

## 7. Verify the boot-chain layer on one VM

```bash
ssh <site-vm>
sudo systemctl status dnsmasq nginx
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost/pxe-map/   # -> 403 (autoindex off) or 404, not connection refused
tftp localhost -c get boot.ipxe /tmp/boot.ipxe && cat /tmp/boot.ipxe   # needs the `tftp` client package (not installed by the role)
ls -Z /var/www/pxe-map /var/lib/tftpboot
# -> httpd_sys_content_t on /var/www/pxe-map, tftpdir_rw_t on /var/lib/tftpboot
sudo firewall-cmd --list-all
# -> tftp and http services, and the 50000-100/udp port range, all present
```

## 8. Verify the API layer on one VM

```bash
ssh <site-vm>
TOKEN=$(sudo grep PXE_MAP_TOKEN /etc/pxe-map-api/env | cut -d= -f2)
PORT=$(sudo grep PXE_MAP_BIND /etc/pxe-map-api/env | cut -d: -f2)

curl -sS localhost:${PORT}/healthz
# -> {"status":"ok"}

curl -sS -X PUT localhost:${PORT}/pxe-map/AA:BB:CC:DD:EE:01 \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"ipxe_url":"http://example.invalid/ipxe-script"}'
# -> 200 {"mac":"aabbccddee01","ipxe_url":"..."}

cat /var/www/pxe-map/aabbccddee01.ipxe
# -> #!ipxe
#    chain http://example.invalid/ipxe-script

ls -Z /var/www/pxe-map/aabbccddee01.ipxe
# -> should show httpd_sys_content_t

curl -sS http://localhost/pxe-map/aabbccddee01.ipxe
# -> confirms nginx (from the pxe_server role) can serve what the API wrote

curl -sS -X DELETE localhost:${PORT}/pxe-map/aabbccddee01 \
  -H "Authorization: Bearer $TOKEN"
# -> 204, and the file is gone
```

If the `ls -Z` context isn't `httpd_sys_content_t`, the SELinux fcontext
rule didn't take - rerun the playbook and check
`sudo ausearch -m avc -ts recent` for denials.

The checks above all run on the VM itself, so they pass even with the API
port closed to the outside. Check it from where the API is actually called
from - a node of this site's MCE cluster (e.g. `oc debug node/<node>`):

```bash
sudo firewall-cmd --list-rich-rules        # on the VM: one rule per allowed source
curl -sS http://<site-vm>:8080/healthz     # from the MCE: {"status":"ok"}, not a timeout
```

Also: `sudo journalctl -u pxe-map-api -f` to tail logs while testing - see
the app repo's `app/main.py` request-logging middleware for what gets
logged per request.

## 9. Point the central BMH-creation endpoint at it

Give the central endpoint's operators, per site:

- That VM's reachable address and port (`PXE_MAP_BIND`).
- The token from `inventory/host_vars/<site>/vault.yml`, delivered
  out-of-band (not over chat/email in plaintext if you can avoid it) -
  `ansible-vault view inventory/host_vars/<site>/vault.yml` to read it.

They call each site's service directly; there's nothing further to
configure here. This service does not do site-selection - that's the
caller's job.

## Upgrading / rotating a token

```bash
git -C ../pxe-map-api pull   # pick up app code changes first
ansible-playbook site.yml --ask-vault-pass                     # both roles, all 5 sites
ansible-playbook site.yml --ask-vault-pass --limit pxe-site3   # both roles, just one site
```

Re-copies `app/`, reinstalls deps, re-renders the env/unit files, and
restarts the service only if something actually changed. To rotate a
site's token: `ansible-vault edit inventory/host_vars/<site>/vault.yml`,
then rerun the playbook for that site.

**InfraEnv boot entries (pxe-map-api `7354800` and later)** change BOTH
layers, so run both roles (the default above, no `--tags`): `pxe_server`
re-templates `boot.ipxe` to look up every NIC's MAC (install-server now maps
only the BareMetalHost's boot MAC), and `pxe_map_api` deploys the app that
writes `<mce_cluster>/<infra_env>.ipxe` boot entries, the conditional
`DELETE ?mce_cluster=` remove-server relies on, and `0644` files nginx can
read. Until a site's VM runs it, IPMI installs from that site's MCE fail
with a 422 from the old API.

## Uninstalling

Not automated (removal is rare and destructive enough to want a human
doing it deliberately, not a playbook flag). On the VM:

```bash
# pxe_map_api layer:
sudo systemctl disable --now pxe-map-api
sudo rm /etc/systemd/system/pxe-map-api.service
sudo systemctl daemon-reload
sudo rm -rf /opt/pxe-map-api /etc/pxe-map-api

# pxe_server layer (only if decommissioning the whole VM - this breaks PXE
# boot for every host at this site):
sudo systemctl disable --now dnsmasq nginx
sudo rm -f /etc/nginx/conf.d/pxe-map.conf /etc/dnsmasq.conf
sudo rm -rf /var/lib/tftpboot

# /var/www/pxe-map and its SELinux fcontext rule are left in place
# deliberately by both of the above - it holds the source-of-truth files.
# Remove manually if you're decommissioning the whole VM:
#   sudo semanage fcontext -d "/var/www/pxe-map(/.*)?"
#   sudo semanage fcontext -d "/var/lib/tftpboot(/.*)?"
#   sudo rm -rf /var/www/pxe-map
#   sudo userdel pxe-map-api
```
