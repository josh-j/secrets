# Secrets Repository

This repository contains encrypted secrets managed by [SOPS](https://github.com/getsops/sops) (Secrets OPerationS) for use across multiple NixOS/Darwin hosts.

## Structure

- `common.yaml` - Shared secrets (API keys, misc public email/name, inter-host SSH keypair)
- `servers.yaml` - Server-only secrets (admin/db/Grafana/Cloudflare/service keys)
- `personal.yaml` - Personal-only secrets (personal email, etc.)
- `hosts/<hostname>.yaml` - Host-specific secrets (optional; none yet)
- `.sops.yaml` - SOPS policy (recipients + creation rules per file)
- `secrets.yaml` - Legacy monolith (kept temporarily for rollback)
- `age-private-key` - Legacy Age key (will be removed once all hosts rely on SSH keys)
- `inter_host`, `config.inter-host` - Legacy SSH key/config (remove after rollout)

## Encryption Keys

The secrets are encrypted for multiple recipients defined in `.sops.yaml`:

1. **Primary age key** (`age1q69cmw...`) - Personal age key for manual operations
2. **deck00 SSH key** (`age1hm3acg...`) - Server host's SSH key
3. **sio-wpc SSH key** (`age1tp2erh...`) - WSL host's SSH key

## Key Model & Rekeying

- NixOS hosts derive their Age identity from `/etc/ssh/ssh_host_ed25519_key` (no `age-private-key` file). Darwin/WSL use `~/.config/sops/age/keys.txt` and must have their public key added to `.sops.yaml`.
- Rekey per file after recipient changes:
  ```bash
  sops updatekeys common.yaml
  sops updatekeys servers.yaml
  sops updatekeys personal.yaml
  # hosts/<name>.yaml as needed
  ```
- Use `ssh-to-age` to derive recipients from SSH host keys. Example:
  ```bash
  ssh-keyscan -t ed25519 <host-or-ip> | ssh-to-age
  ```

## Usage in NixOS/Darwin

- NixOS uses `sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];` (no key file).
- Darwin/WSL use `~/.config/sops/age/keys.txt` as `sops.age.keyFile`.
- Default secrets file: `servers.yaml`; per-secret overrides for common/personal scopes.
- SSH config/known_hosts are generated from `hosts/hosts.yaml`; inter-host key is read from `config.sops.secrets."ssh_inter_host_private_key".path`.

## Security Notes

- Never commit plaintext secrets; keep private keys off-repo.
- Secrets decrypt to `/run/secrets/` with restricted permissions.
- Legacy files (`secrets.yaml`, `age-private-key`, `inter_host`, `config.inter-host`) remain only for rollback; remove after full rollout.

## TODO

- [ ] Add `maas` (desktop) and `needl` (Darwin) Age recipients and rekey scoped files.
- [ ] Automate `.sops.yaml` recipient generation from `hosts/hosts.yaml` (with dry-run).
- [ ] Document required vs optional secrets per host role and add host-specific files as needed.
- [ ] Back up and then remove legacy plaintext artifacts (`age-private-key`, `inter_host`, `config.inter-host`) post-rollout.
- [ ] Add recovery/offline Age key with minimal scope and document recovery steps.
