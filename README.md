# Secrets Repository

This repository contains encrypted secrets managed by [SOPS](https://github.com/getsops/sops) (Secrets OPerationS) for use across multiple NixOS/Darwin hosts.

## Structure

- `secrets.yaml` - Main encrypted secrets file containing API keys, credentials, and configuration
- `.sops.yaml` - SOPS configuration defining which age keys can decrypt the secrets
- `age-private-key` - Primary age private key for manual decryption
- `inter_host` - SSH keys for inter-host communication
- `config.inter-host` - SSH configuration for inter-host access

## Encryption Keys

The secrets are encrypted for multiple recipients defined in `.sops.yaml`:

1. **Primary age key** (`age1q69cmw...`) - Personal age key for manual operations
2. **deck00 SSH key** (`age1hm3acg...`) - Server host's SSH key
3. **sio-wpc SSH key** (`age1tp2erh...`) - WSL host's SSH key

## Re-encrypting Secrets

When you add a new host to your NixOS configuration, you must re-encrypt `secrets.yaml` to include that host's age key. This ensures the host can decrypt the secrets during system activation.

### Why Re-encryption Was Needed

On 2025-11-16, the `sio-wpc` (WSL) host was added to the NixOS configuration with `siof.system.secrets.enable = true`. During system activation, the following error occurred:

```
sops-install-secrets: failed to decrypt 'secrets.yaml': Error getting data key: 0 successful groups required, got 0
```

**Root cause:** The `secrets.yaml` file was previously encrypted only for the primary and deck00 keys. Even though the `sio-wpc_ssh` key was defined in `.sops.yaml`, the actual `secrets.yaml` file hadn't been re-encrypted to include it.

### How to Re-encrypt

When adding a new host:

1. Add the host's age key to `.sops.yaml` under `keys:` and include it in the appropriate `key_groups`
2. Run the following command from this directory:
   ```bash
   sops updatekeys secrets.yaml
   ```
3. Confirm when prompted to add the new key
4. Commit the updated `secrets.yaml` to the repository

### Getting a Host's Age Key

For NixOS hosts using SSH host keys:

```bash
# On the target host
cat /etc/ssh/ssh_host_ed25519_key.pub | ssh-to-age
```

For Darwin hosts or manual age keys:

```bash
# Generate a new age key
age-keygen -o ~/.config/sops/age/keys.txt
# Extract the public key from the generated file
grep "public key:" ~/.config/sops/age/keys.txt
```

## Usage in NixOS Configuration

Secrets are automatically decrypted during system activation by the `sops-nix` module. The configuration is defined in `modules/common/secrets.nix`.

Individual secrets are accessed via:
- `config.sops.secrets."secret_name".path` - Path to decrypted secret file
- `config.sops.placeholder."secret_name"` - Placeholder for use in templates

## Security Notes

- Never commit unencrypted secrets to version control
- Keep age private keys secure and backed up separately
- Each host only receives the decryption key it needs (via SSH host keys)
- Secrets are decrypted at activation time and stored in `/run/secrets/` with restricted permissions

## TODO

- [ ] Add `maas` (desktop) host SSH key to `.sops.yaml` and re-encrypt
- [ ] Add `needl` (Darwin laptop) age key to `.sops.yaml` and re-encrypt
- [ ] Document which secrets are required vs optional for each host type (desktop/server/WSL)
- [ ] Set up automatic key rotation schedule for critical secrets (API keys, passwords)
- [ ] Create backup procedure for age private keys
- [ ] Consider separating secrets into multiple files (e.g., `secrets-server.yaml`, `secrets-desktop.yaml`)
- [ ] Add pre-commit hook to verify `.sops.yaml` and `secrets.yaml` are in sync before commits
- [x] ~~Document the GitHub token in flake.nix (line 58) and determine if it should be moved to secrets~~ - Fixed: Token removed from flake.nix, now using netrc with SOPS-managed `api_github` secret
