# cyberark_auth

Ansible role that authenticates to CyberArk Identity using OAuth2 client credentials and exposes the resulting bearer token as `cyberark_token` on all hosts in the play.

## How it works

1. Validates the three required variables are set
2. POSTs to the CyberArk Identity platform token endpoint from `localhost` (once per play)
3. Sets `cyberark_token` on every host via `set_fact cacheable: true`, making it available to all subsequent tasks and plays in the same run

## Requirements

- Ansible 2.9+
- A CyberArk Identity tenant with an OAuth2 service user (client credentials grant)

## Role variables

All three variables are required and have no default — the role will fail fast with a clear message if any are unset.

| Variable | Description |
|---|---|
| `cyberark_identity_tenant` | Identity tenant ID — forms the subdomain `<tenant>.id.cyberark.cloud` |
| `cyberark_client_id` | OAuth2 client ID for the service user |
| `cyberark_client_secret` | OAuth2 client secret — **store in Ansible Vault** |

## Output

| Variable | Description |
|---|---|
| `cyberark_token` | Bearer token set on all hosts; use as `Authorization: Bearer {{ cyberark_token }}` |

## Token lifetime

CyberArk Identity tokens expire after **~1 hour** (the exact TTL is returned in `expires_in` on the auth response but is not exposed by this role). Be aware of two scenarios:

- **Long-running plays** — if a play takes longer than the token TTL, API calls late in the run will fail with 401. Break long plays into shorter runs and include this role at the top of each.
- **Fact cache across plays** — `cacheable: true` persists `cyberark_token` to the Ansible fact cache. A subsequent play that hits the cache will silently use a stale token if the cache TTL outlives the token TTL. Either set your fact cache TTL below 3600 seconds, or explicitly include this role at the start of every play rather than relying on the cache.

## Usage

Install the role:

```yaml
# requirements.yml
roles:
  - name: cyberark_auth
    src: https://github.com/TobyAnscombe/cyberark-api-management
    version: main
```

```bash
ansible-galaxy install -r requirements.yml
```

Call the role from your playbook and use the token in subsequent tasks:

```yaml
- name: My CyberArk playbook
  hosts: all
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    # cyberark_client_id and cyberark_client_secret from vault:
    # ansible-vault encrypt_string '...' --name cyberark_client_id

  roles:
    - cyberark_auth

  tasks:
    - name: List accounts in a safe
      uri:
        url: "https://mysubdomain.privilegecloud.cyberark.cloud/passwordvault/api/Accounts?safeName=my-safe"
        method: GET
        headers:
          Authorization: "Bearer {{ cyberark_token }}"
      delegate_to: localhost
      run_once: true
```

## Vault example

```bash
# Encrypt secrets into a vault file
ansible-vault encrypt_string 'my-client-id' --name cyberark_client_id
ansible-vault encrypt_string 'my-client-secret' --name cyberark_client_secret
```

Pass the vault file at runtime:

```bash
ansible-playbook site.yml --ask-vault-pass
# or
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

## License

MIT
