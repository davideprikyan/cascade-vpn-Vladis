# Cascade VPN

Ansible playbooks for configuring cascade VPN servers.

## Architecture

VPN clients → tunnel-in-tunnel machine (`amnezia-wireguard`) → `awg0` → Remote AWG server → internet via remote's IP.

Policy routing on the tunnel-in-tunnel host forces all VPN client traffic through the remote tunnel instead of tunnel-in-tunnel machine internet connection.

## Prerequisites

- Ansible installed locally
- SSH access to the target hosts (keys configured)
- `ansible-vault` for managing secrets

## Repository structure

```
inventory.ini                     # host inventory
tunnel_cascade.yml                # main playbook
host_vars/vk-tunnel-in-tunnel.yml # per-host variables (non-sensitive)
vars.yml                          # vault-encrypted secrets
templates/                        # Jinja2 config templates
files/                            # static files deployed as-is
```

## Setup

### 1. Create the vault secrets file

The playbook requires a `vars.yml` file with the following vault-encrypted variables:

```bash
ansible-vault create vars.yml
```

Add the following content:

```yaml
vault_awg_client_private_key: "..."
vault_awg_server_public_key: "..."
vault_awg_preshared_key: "..."
vault_awg_server_endpoint: "host:port"
```

Values can be found in the configuration you create for new user in the AmneziaVPN client.

### 2. Verify connectivity

```bash
ansible all -i inventory.ini -m ping
```

## Running the playbook

Deploy the cascade VPN tunnel on `vk-tunnel-in-tunnel`:

```bash
ansible-playbook -i inventory.ini tunnel_cascade.yml --ask-vault-pass
```

Or with a vault password file:

```bash
ansible-playbook -i inventory.ini tunnel_cascade.yml --vault-password-file ~/.vault_pass
```

## Checking tunnel status on the server

```bash
ssh ubuntu@<tunnel-in-tunnel-ip>
sudo systemctl status amnezia-awg-client
sudo docker logs amnezia-awg-client
sudo ip rule show
sudo ip route show table 100
```

## How it works

The playbook configures a cascade VPN on `vk-tunnel-in-tunnel`:

1. Deploys an AmneziaWG client tunnel (`awg0`) to the remote server using `amneziavpn/amneziawg-go:latest` Docker image
2. Runs the container under a systemd service (`amnezia-awg-client.service`) with `--network host` so the tunnel interface appears on the host
3. Sets up policy routing so all VPN client traffic exits through remote server instead of your macine's internet connection
4. Adds an iptables exception so the WireGuard server (for laptop connections) still responds directly
