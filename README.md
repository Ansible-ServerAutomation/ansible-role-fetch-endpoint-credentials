# ansible-role-fetch-endpoint-credentials

An Ansible role that runs on **localhost** (AWX Tower) and dynamically attaches the correct AWX credential to the currently running job template before any subsequent roles execute.

It works by inspecting the **host variables** of a target endpoint (identified by `inventory_host`) for a variable whose name ends with `_credential`. The value of that variable is treated as an AWX credential name, looked up via the AWX REST API, and then attached to the running job template using the `AWX_JOB_TEMPLATE_ID` environment variable that AWX automatically injects at runtime.

---

## Requirements

| Requirement | Version |
|---|---|
| Ansible | >= 2.9 |
| AWX / Ansible Tower | Any version supporting `/api/v2/` |

No additional collections or Python packages are required. All AWX API interactions use Ansible's built-in `ansible.builtin.uri` module.

---

## Role Variables

### Variables with defaults (`defaults/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `awx_host` | `https://localhost` | Base URL of the AWX Tower instance (no trailing slash). |
| `awx_validate_certs` | `true` | Whether to validate the AWX API SSL certificate. Set to `false` only in lab/test environments. |
| `credential_variable_suffix` | `_credential` | Suffix used to identify the credential variable in the endpoint's host variables. |

### Required variables (no defaults — must be supplied at runtime)

| Variable | Description |
|---|---|
| `awx_token` | AWX OAuth2 Bearer token used to authenticate REST API calls. Pass via a custom AWX credential or as an encrypted extra variable. **Never commit this value to source control.** |
| `inventory_host` | The hostname of the target endpoint **exactly as it appears in the AWX inventory**. Set as an extra variable in the AWX job template, typically driven by the job's **Limit** field. |

### Variables set internally during execution (not for external use)

| Variable | Description |
|---|---|
| `_credential_var_keys` | List of host variable keys ending with `_credential` (expected to contain exactly one entry). |
| `_endpoint_credential_name` | The AWX credential name read from the matched host variable. |
| `_awx_credential_id` | The numeric AWX credential ID returned by the API lookup. |
| `_awx_job_template_id` | The AWX job template ID read from the `AWX_JOB_TEMPLATE_ID` environment variable. |

---

## How It Works

```
AWX Job Template runs
        │
        ▼
[Role: fetch-endpoint-credentials]  (executes on localhost)
        │
        ├─ 1. Reads hostvars[inventory_host] from AWX inventory
        │
        ├─ 2. Finds variable ending with '_credential'
        │       e.g.  oracle_db_credential: "PROD-Oracle-SvcAcct"
        │
        ├─ 3. Calls GET /api/v2/credentials/?name=PROD-Oracle-SvcAcct
        │       → retrieves credential ID (e.g. 42)
        │
        ├─ 4. Reads AWX_JOB_TEMPLATE_ID env var (auto-injected by AWX)
        │
        └─ 5. Calls POST /api/v2/job_templates/{id}/credentials/
                  body: {"id": 42}
                → HTTP 204: credential attached ✓
        │
        ▼
[Subsequent roles run with the credential available]
```

---

## Host Variable Convention

On each managed endpoint in the AWX inventory, add **exactly one** host variable whose name ends with `_credential`. The value must be the **exact name** of an existing AWX credential.

**Example** (host_vars for `db-server-01`):

```yaml
# AWX inventory host_vars / host variables for db-server-01
oracle_db_credential: "PROD-Oracle-DB-ServiceAccount"
```

The role will fail with a descriptive error if:
- Zero variables ending with `_credential` are found.
- More than one variable ending with `_credential` is found.
- The credential name value is empty.
- The named credential does not exist in AWX.

---

## AWX Job Template Setup

1. **Inventory**: Attach an inventory that contains `inventory_host` as a host, with the appropriate `_credential` host variable set.

2. **Limit**: Set the job template's **Limit** field (or pass it as a prompt) to the target hostname. This ensures `inventory_host` resolves correctly.

3. **Extra Variables**: Add the following extra variables to the job template:

   ```yaml
   inventory_host: "{{ awx_job_hosts | first }}"
   ```

   Or pass `inventory_host` explicitly when launching the job:

   ```yaml
   inventory_host: "db-server-01"
   ```

4. **Credentials**: Attach a custom AWX credential (or use an encrypted extra variable) to supply `awx_token` to the playbook.

5. **Playbook order**: Place this role **first** in the play so the credential is attached before any subsequent roles execute.

---

## Example Playbook

```yaml
---
- name: Fetch endpoint credential and run automation
  hosts: localhost
  gather_facts: false

  roles:
    - role: ansible-role-fetch-endpoint-credentials
      vars:
        awx_host: "https://awx.example.com"
        awx_token: "{{ vault_awx_token }}"
        inventory_host: "{{ target_host }}"

    # Subsequent roles will now have the correct credential attached
    - role: my-other-automation-role
```

---

## Generating an AWX OAuth2 Bearer Token

The `awx_token` variable must be an AWX personal access token (OAuth2 Bearer token). The following methods can be used to generate one.

### Method 1: AWX Web UI

1. Log in to the AWX web interface.
2. Click your username in the top-right corner and select **User Details**, or navigate to **Users** → select your user.
3. Select the **Tokens** tab.
4. Click **Add** (the `+` button).
5. Fill in the form:
   - **Application**: Leave blank for a personal access token (not tied to an OAuth2 application).
   - **Description**: Enter a meaningful description (e.g., `ansible-role-fetch-endpoint-credentials`).
   - **Scope**: Select **Write** to allow the token to attach credentials to job templates.
6. Click **Save**.
7. Copy the token value immediately — it is shown **only once**.

> **Note:** Store the token securely. Use AWX credential injection or Ansible Vault rather than storing it in plain text.

---

### Method 2: AWX REST API

Send a `POST` request to `/api/v2/tokens/` using your AWX username and password for Basic Auth:

```bash
curl -s -X POST https://<awx-host>/api/v2/tokens/ \
  -H "Content-Type: application/json" \
  -u "<username>:<password>" \
  -d '{"description": "ansible-role-fetch-endpoint-credentials", "application": null, "scope": "write"}' \
  | python3 -m json.tool
```

The response will contain a `token` field — copy that value and store it securely.

---

### Method 3: AWX CLI (`awx` command-line tool)

If the [AWX CLI](https://docs.ansible.com/automation-controller/latest/html/controllercli/index.html) is installed and configured:

```bash
awx login --conf.host https://<awx-host> \
           --conf.username <username> \
           --conf.password <password>
```

This writes a token to `~/.config/tower-cli.cfg`. You can also create a named token directly:

```bash
awx tokens create \
  --description "ansible-role-fetch-endpoint-credentials" \
  --scope write \
  -f human
```

---

### Supplying the Token to the Role

Once generated, provide `awx_token` to the role using one of these approaches (in order of preference):

| Approach | How |
|---|---|
| **AWX Custom Credential** | Create a credential type that injects `awx_token` as an extra variable, then attach it to the job template. |
| **Ansible Vault** | Encrypt the token with `ansible-vault encrypt_string` and reference it as `vault_awx_token`. |
| **AWX Encrypted Extra Variable** | Mark the extra variable as sensitive in the job template's **Extra Variables** field. |

**Never commit the raw token value to source control.**

---

## Security Considerations

- `awx_token` is marked `no_log: true` on every task that uses it. It will **not** appear in AWX job output or logs.
- Do not store `awx_token` in plain text in inventory or extra variables. Use AWX credential injection or Ansible Vault.
- Set `awx_validate_certs: true` (the default) in production environments to prevent MITM attacks against the AWX API.
- Revoke tokens that are no longer needed via **Users** → **Tokens** in the AWX UI, or via `DELETE /api/v2/tokens/<id>/`.

---

## Dependencies

None. This role has no external role or collection dependencies.

---

## License

MIT

---

## Author

[Ansible-ServerAutomation](https://github.com/Ansible-ServerAutomation)

