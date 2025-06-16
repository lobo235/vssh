# vssh - Secure SSH Access via Vault-signed Certificates

`vssh` is a bash script that simplifies secure, short-lived SSH access using [HashiCorp Vault's SSH Secret Engine](https://developer.hashicorp.com/vault/docs/secrets/ssh). It automatically manages certificate signing and renewal, enabling a seamless experience for users who need strong authentication with ephemeral credentials.

Created by [Justin Barlow (netlobo)](https://github.com/lobo235), this script is ideal for technical users, homelab enthusiasts, and professionals seeking to automate SSH certificate authentication.

---

## ✨ Features

* Automatically signs SSH public key using Vault
* Validates existing cert expiration and renews if needed
* Securely logs actions with optional color-coded output
* Works seamlessly for both interactive and non-interactive SSH sessions
* Easy to customize through environment variables or `.bashrc` overrides

---

## 🌐 Prerequisites

* Bash 4.0+
* `ssh`, `ssh-keygen`, and `vault` CLI in your `$PATH`
* A valid Vault token is available via `VAULT_ADDR` and Vault auth (e.g., userpass, approle, etc.)
* Your Vault user has permission to use the `ssh/sign/<role>` endpoint
* Vault's SSH Secret Engine must be enabled and configured
* Remote hosts must trust the Vault SSH CA

Useful links:

* [Vault SSH Secrets Engine Overview](https://developer.hashicorp.com/vault/docs/secrets/ssh)
* [Vault CLI Docs](https://developer.hashicorp.com/vault/docs/commands)
* [Vault Policies](https://developer.hashicorp.com/vault/docs/concepts/policies)
* [SSH CA Certificates with Vault](https://developer.hashicorp.com/vault/tutorials/secrets-management/ssh-certificate-authentication)

---

## 🚀 Installation

1. **Clone the repo or download the script**

```bash
git clone https://github.com/lobo235/vssh.git
cd vssh
cp vssh ~/bin/vssh && chmod +x ~/bin/vssh
```

2. **Add to PATH**

Ensure `~/bin` is in your PATH (add to `.bashrc` if needed):

```bash
export PATH="$HOME/bin:$PATH"
```

> **Note**: To make `vssh` system-wide, place it in `/usr/local/bin` instead (requires sudo).

---

## ⚖️ Default Configuration & Assumptions

The script uses the following defaults, all of which can be overridden via environment variables or `.bashrc`:

| Variable                 | Default                                                                            | Description                 |
| ------------------------ | ---------------------------------------------------------------------------------- | --------------------------- |
| `VSSH_VAULT_ROLE`        | `$(whoami)`                                                                        | Vault SSH role name         |
| `VSSH_PRIVATE_KEY`       | Most common default private key in `~/.ssh/` (prefers `id_ed25519`, then `id_rsa`) | SSH private key path        |
| `VSSH_PUBLIC_KEY`        | `${VSSH_PRIVATE_KEY}.pub`                                                          | SSH public key path         |
| `VSSH_CERT_PATH`         | `${VSSH_PRIVATE_KEY}-cert.pub`                                                     | Output cert path            |
| `VSSH_SIGN_PATH`         | `ssh/sign/${VSSH_VAULT_ROLE}`                                                      | Vault endpoint path         |
| `VSSH_RENEW_BUFFER_SECS` | `300` (5 min)                                                                      | Time before expiry to renew |

---

## 🔧 Usage

### Basic SSH

```bash
vssh <hostname>
```

### Run remote command

```bash
vssh <hostname> uptime
```

### Pipe input (MOTD may display)

```bash
echo "uptime" | vssh <hostname>
```

---

## 🪧 Setting Up Vault SSH Secret Engine

### 1. Enable the secret engine

```bash
vault secrets enable -path=ssh ssh
```

More info: [Enable Secrets Engines](https://developer.hashicorp.com/vault/docs/secrets)

### 2. Generate or upload your SSH CA key

You have two options:

#### Option A: Let Vault generate the CA key (recommended for simplicity and security)

```bash
vault write -f ssh/config/ca
```

This will create a new key pair internally. The private key remains secured inside Vault, making this the most secure and operationally simple option. Avoids risks associated with external key reuse.

To retrieve the public CA key for distribution to your hosts:

```bash
vault read -field=public_key ssh/config/ca > ssh_ca.pub
```

You can also use the Vault UI to generate the key under the SSH Secrets Engine > Configuration > Generate CA.

More info: [Vault SSH CA Config](https://developer.hashicorp.com/vault/docs/secrets/ssh/signed-ssh-certificates#configure-vault-as-a-ca)

#### Option B: Upload your own existing CA public key

```bash
vault write ssh/config/ca public_key=@/path/to/your/ssh_ca.pub
```

> ⚠️ Use this option only if absolutely necessary. Before uploading an existing CA key:
>
> * Ensure it was never compromised or used in less-secure environments.
> * Audit who or what systems had access to the corresponding private key.
> * Consider rotating to a Vault-generated CA as soon as possible.

### 3. Create a signing role

```bash
vault write ssh/roles/<role_name> \
  key_type=ca \
  allow_user_certificates=true \
  allowed_users="*" \
  default_extensions="permit-pty,permit-port-forwarding" \
  ttl="60m"
```

> 🔒 **Security Tip**: The `ssh/roles/*` paths should be protected with Vault policies that limit access to authorized users or groups only. This prevents abuse of cert signing capabilities.
>
> Refer to: [Vault Policy Documentation](https://developer.hashicorp.com/vault/docs/concepts/policies)

### 🔐 Example Vault Policies

#### For a specific user `bob` and Vault SSH role `bob`

```hcl
path "ssh/sign/bob" {
  capabilities = ["update"]
}
```

#### For a group `devops` using Vault SSH role `team-devops`

```hcl
path "ssh/sign/team-devops" {
  capabilities = ["update"]
}
```

Assign this policy to an identity group (e.g., `devops`) via Vault's Identity system.

---

## 🔐 Configure Remote Hosts (sshd)

1. **Install the CA public key on each host**

```bash
sudo cp ssh_ca.pub /etc/ssh/trusted-user-ca-keys.pem
```

2. **Update `sshd_config`**

```ini
TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem
```

3. **Restart SSH**

```bash
sudo systemctl restart sshd
```

4. **Disable regular pubkey access if needed**

To force all SSH access through signed certs:

```ini
PubkeyAuthentication no
```

> ⚠️ Be aware: Even if you're using certificate-based authentication, any existing public keys listed in `~/.ssh/authorized_keys` will still work **unless** you disable `PubkeyAuthentication`. This means users who previously authenticated via standard pubkey can still gain access. Use this setting if you want to enforce strict CA-only login.

More info: [OpenSSH CA Authentication](https://man.openbsd.org/sshd_config#TrustedUserCAKeys)

---

## 🚫 What May Not Work (and Why)

### Using `echo "cmd" | vssh <host>` suppressing output

* This is expected: SSH with piped input does not allocate a TTY and may show MOTD. `vssh` doesn't try to change this default behavior to maintain compatibility with `ssh`.

### Using aliases instead of a script

* Aliases don’t play well with piped input or command substitution. `vssh` must be a real script to behave correctly in all contexts.

### Cert not renewing properly

* Some users may find that their cert appears to renew every time they run `vssh`. This is often caused by inconsistent `ssh-keygen` output formats across different platforms (e.g., OpenSSH on macOS vs Linux).
* The script relies on parsing the `Valid:` line from `ssh-keygen -L -f <cert>` output. If your local SSH version formats this line differently, it may fail to extract the expiration timestamp.

#### 🔧 Troubleshooting Steps:

1. Run manually and check output:

```bash
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub
```

2. Look for the line beginning with `Valid:` — if the format deviates from:

```
Valid: from YYYY-MM-DDTHH:MM:SS to YYYY-MM-DDTHH:MM:SS
```

the regex in the script may not parse it correctly.

3. To debug, set `set -x` in the script or echo parsed variables.

4. Consider upgrading OpenSSH to a more recent version if output format seems unusual.

---

## 🪡 Pro Tips

* You can simulate expiration to test renewal logic by moving or deleting the cert:

```bash
mv ~/.ssh/id_ed25519-cert.pub ~/.ssh/id_ed25519-cert.pub.bak
vssh beast
```

* Or force early renewal by setting the buffer high (assumes your Vault SSH role TTL is 1 hour):

```bash
export VSSH_RENEW_BUFFER_SECS=3600
vssh beast
```

> Adjust `3600` depending on the `ttl` configured in your Vault SSH role.

* To debug, increase verbosity:

```bash
vssh -vvv beast
```

---

## 📃 Logging Behavior

* Logs are printed to `stderr` so `stdout` remains clean and usable in scripts.
* Cert creation, renewal, and expiration status are clearly logged.

---

## 📖 License

MIT License — see [LICENSE](LICENSE) file for details.

---

## 👤 Author

Justin Barlow — [@lobo235](https://github.com/lobo235)
