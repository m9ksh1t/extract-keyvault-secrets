# Extract Azure Key Vault secrets via GitHub Actions

Two-workflow setup: a **caller** workflow you trigger, which invokes a
**reusable** workflow that logs into Azure, reads `config/secrets-list.json`
(the names of secrets that already exist in the vault), fetches each
one's current value, and updates that same file **in place** on the
runner's checked-out copy, no second file is created. The updated file
is then uploaded as a short-lived build artifact so you can download it
and check the values. Nothing is committed back to the repository; the
copy in git still has the blank/placeholder values after the run.

## Files

- `.github/workflows/extract-secrets.yml` - caller workflow (`workflow_dispatch`)
- `.github/workflows/reusable-keyvault-extract.yml` - reusable workflow (`workflow_call`); 
- `config/secrets-list.json` - example input: the *names* of secrets to fetch (values left blank)
- `config/upsert-secrets.json` - example input: the key-value pairs of secrets which have to be updated or inserted

## One-time setup

This uses **OIDC federated credentials** for the Azure login - GitHub, so there's no client secret to store or rotate.

### 1. Azure app registration and federated credentials

Create the app registration + service principal and grant the required roles it needs to extract, insert or update secrets into the Azure Keyvault (User and Service principal). Also add the federated credentials for your specific repo.
Roles:
 - Key Vault Secrets Officer
 - Key Vault Secrets User

### 2. Store IDs as GitHub secrets

In the repo (or an **Environment**, recommended so you can require
reviewer approval before the job runs) → Settings → Secrets and
variables → Actions, add:

- `AZURE_CLIENT_ID` — the app's `appId`
- `AZURE_TENANT_ID` — your Azure AD tenant ID
- `AZURE_SUBSCRIPTION_ID` — the subscription containing the vault

Storing them as Actions secrets keeps them out of the workflow YAML and out of logs.

### 3. List the secret names you want

Edit `config/secrets-list.json` (or add another file at any path) with
the names of secrets that already exist in the vault, e.g.:

```json
{
  "DatabasePassword": "",
  "UserId": ""
}
```

The values are ignored on input - only the **keys** matter, they are the
secret names looked up in Key Vault. If a named secret doesn't exist in
the vault (or the service principal can't read it), that key is left
with an empty string `""` rather than failing the whole run - a
`::warning::` is logged for each one, and a summary warning lists every
missing name at the end of the step.

### 4. Run it

Actions tab -> "Extract Secrets" -> Run workflow -> fill in `keyvault-name`
(and `secrets-file` if not using the default path). The job checks out
the repo, updates `config/secrets-list.json` in place on that checkout
(the file on disk in the running job, not the file in git), and uploads
it as the `extracted-secrets` artifact. Open the run -> Artifacts ->
download `extracted-secrets` to see:

```json
{
  "DatabasePassword": "admin123",
  "UserId": "Admin"
}
```
