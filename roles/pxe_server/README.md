# pxe_server

Provisions the TFTP + HTTP boot-chain prerequisites for a site PXE VM: iPXE
firmware served over TFTP by `dnsmasq` (TFTP-only mode - no DNS, no DHCP),
and `nginx` serving `/var/www/pxe-map/` read-only over HTTP. This is the
"layer 1" role - it does not install or manage `pxe-map-api` itself (see the
sibling `pxe_map_api` role for that).

## Boot chain this role supports

1. Server firmware DHCPs (external Windows DHCP, not managed by this role or
   this repo). Gets `next-server` = this VM, `filename` = `ipxe.efi`.
2. Firmware TFTPs `ipxe.efi` from this VM and runs it.
3. iPXE re-DHCPs with option 77 = `iPXE`. Windows DHCP policy returns
   `filename` = `boot.ipxe`.
4. iPXE TFTPs `boot.ipxe` from this VM. That file chains over HTTP to
   `/pxe-map/<mac>.ipxe` on this VM, where `<mac>` is the booting NIC's MAC as
   12 lowercase hex chars, no separators.
5. That per-MAC file chains to the host's InfraEnv iPXE script on the hub
   cluster (owned by `pxe-map-api`, not this role).

## Variables (see `defaults/main.yml`)

| Variable | Default | Notes |
|---|---|---|
| `pxe_server_vm_ip` | `CHANGE_ME` | This VM's provisioning-facing IP, templated into `boot.ipxe`'s chain URL. Must be set per site. |
| `pxe_server_interface` | `CHANGE_ME` | NIC dnsmasq binds to. Must be set per site. |
| `pxe_server_tftp_root` | `/var/lib/tftpboot` | |
| `pxe_server_pxe_map_dir` | `/var/www/pxe-map` | Same literal path as `pxe_map_api_dir` in the `pxe_map_api` role - override both together if you change it. |
| `pxe_server_tftp_port_range_start` / `_end` | `50000` / `50100` | dnsmasq's TFTP data-connection port range, opened in firewalld. |
| `pxe_server_ipxe_efi_src` | `/usr/share/ipxe/ipxe-x86_64.efi` | Source path on the target host (from the `ipxe-bootimgs-x86` package), copied into the TFTP root. |

## Out of scope (on purpose)

- DHCP configuration - the external Windows DHCP server is out of scope.
- Per-MAC `pxe-map` file management - `pxe-map-api` owns that; this role only
  creates the empty directory.
- Any verification/curl/tftp-get checks - configuration only.
- Disabling or loosening SELinux - it must stay enforcing.

## Shared directory with `pxe_map_api`

Both roles manage `/var/www/pxe-map`: this role creates it owned by `nginx`
(so nginx can serve it before `pxe-map-api` is even installed), while
`pxe_map_api` re-owns it as the `pxe-map-api` service user once that role
runs. `site.yml` always runs `pxe_server` before `pxe_map_api`, so a full
playbook run converges to a stable owner. If you run the two roles
independently via `--tags` and alternate between them on the same host, the
directory's owner will flip between runs - harmless in practice since mode
`0755` gives every process (including nginx) read + traverse regardless of
owner, but worth knowing if `ansible-playbook --check` reports a "change"
you didn't expect.
