# Extract Azure Key Vault secrets via GitHub Actions

Two-workflow setup: a **caller** workflow you trigger, which invokes a
**reusable** workflow that logs into Azure, reads `config/secrets-list.json`
(the names of secrets that already exist in the vault), fetches each
one's current value, and updates that same file **in place** on the
runner's checked-out copy — no second file is created. The updated file
is then uploaded as a short-lived build artifact so you can download it
and check the values. Nothing is committed back to the repository; the
copy in git still has the blank/placeholder values after the run.

## Files

- `.github/workflows/extract-secrets.yml` — caller workflow (`workflow_dispatch`)
- `.github/workflows/reusable-keyvault-extract.yml` — reusable workflow (`workflow_call`); the extraction logic (the `az keyvault secret show` loop) lives inline in its "Extract secrets from Key Vault" step, no separate script file
- `config/secrets-list.json` — example input: the *names* of secrets to fetch (values left blank)

## One-time setup

This uses **OIDC federated credentials** for the Azure login — GitHub
issues a short-lived token for each workflow run and Azure AD trusts it
directly, so there's no client secret to store or rotate.

### 1. Azure app registration

```bash
# Create the app registration + service principal (no --create-cert / -password needed)
az ad app create --display-name "gha-keyvault-reader"
# Note the "appId" from the output — that's AZURE_CLIENT_ID

az ad sp create --id <APP_ID_FROM_ABOVE>
```

Grant it read access to the vault:

```bash
az role assignment create \
  --assignee <APP_ID> \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RG_NAME>/providers/Microsoft.KeyVault/vaults/<VAULT_NAME>
```

If your vault uses the older **access policy** model instead of Azure
RBAC, grant the service principal a **Get** secret permission via
`az keyvault set-policy` instead of the role assignment above.

### 2. Add a federated credential for this repo

This is the step that replaces the client secret — it tells Azure AD to
trust GitHub's OIDC tokens for this specific repo (and, ideally, a
specific branch or environment):

```bash
az ad app federated-credential create \
  --id <APP_ID> \
  --parameters '{
    "name": "github-actions-extract-secrets",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<GITHUB_ORG>/<GITHUB_REPO>:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

- Replace `<GITHUB_ORG>/<GITHUB_REPO>` with your repo.
- The `subject` scopes exactly which trigger context is trusted. A few
  common patterns:
  - Branch: `repo:ORG/REPO:ref:refs/heads/main`
  - Any pull request: `repo:ORG/REPO:pull_request`
  - A GitHub **Environment** named `production`: `repo:ORG/REPO:environment:production`
  - Environments are the safest option if this can touch production
    secrets — pair it with required reviewers.
- You can create multiple federated credentials on the same app (e.g.
  one per environment) if different branches/environments need to
  trigger this.

### 3. Store IDs as GitHub secrets

In the repo (or an **Environment**, recommended so you can require
reviewer approval before the job runs) → Settings → Secrets and
variables → Actions, add:

- `AZURE_CLIENT_ID` — the app's `appId`
- `AZURE_TENANT_ID` — your Azure AD tenant ID (`az account show --query tenantId -o tsv`)
- `AZURE_SUBSCRIPTION_ID` — the subscription containing the vault

None of these are secret in the sense of being sole proof of identity —
only a workflow run whose OIDC token matches the federated credential's
`subject` can actually authenticate — but storing them as Actions
secrets keeps them out of the workflow YAML and out of logs.

### 4. List the secret names you want

Edit `config/secrets-list.json` (or add another file at any path) with
the names of secrets that already exist in the vault, e.g.:

```json
{
  "DatabasePassword": "",
  "UserId": ""
}
```

The values are ignored on input — only the **keys** matter, they are the
secret names looked up in Key Vault. If a named secret doesn't exist in
the vault (or the service principal can't read it), that key is left
with an empty string `""` rather than failing the whole run — a
`::warning::` is logged for each one, and a summary warning lists every
missing name at the end of the step.

### 5. Run it

Actions tab → "Extract Secrets" → Run workflow → fill in `keyvault-name`
(and `secrets-file` if not using the default path). The job checks out
the repo, updates `config/secrets-list.json` in place on that checkout
(the file on disk in the running job, not the file in git), and uploads
it as the `extracted-secrets` artifact. Open the run → Artifacts →
download `extracted-secrets` to see:

```json
{
  "DatabasePassword": "admin123",
  "UserId": "Admin"
}
```

## Security notes — please read before using this in production

- The populated JSON contains **plaintext secret values**. It is
  uploaded as a GitHub Actions artifact with a 1-day retention
  (`artifact-retention-days` input), and only repo collaborators with
  read access can download workflow artifacts — but anyone with that
  access can retrieve the plaintext for as long as the artifact exists.
  If you'd rather nothing persist at all, don't add the upload step and
  instead consume the now-updated `${{ inputs.secrets-file }}` directly
  in a later step of the *same job* (e.g. feed it straight into a deploy
  step), then let the runner get torn down.
- Every fetched value is masked with `::add-mask::` before anything else
  touches it, so it won't appear in plain text in the Actions log even
  if a later step echoes it by accident — but masking is a best-effort
  log filter, not encryption, and can be bypassed by transformations
  (base64, splitting the string, etc.) done in later steps.
- The update happens only on the runner's checked-out copy — there is no
  `git commit` / `git push` step in this workflow, so the populated
  values never land in git history. Do not add a commit-back step for
  this file; committing real secret values into version control defeats
  the point of fetching them at run time.
- Consider scoping the service principal's role assignment to just that
  one Key Vault (as shown above) rather than the whole subscription, and
  putting the job behind a GitHub **Environment** with required
  reviewers if this can run against production secrets.
- There's no long-lived Azure credential stored in GitHub — the OIDC
  federated credential means only workflow runs matching the `subject`
  you configured in step 2 can authenticate as this app, and each login
  token is short-lived. Keep that `subject` as narrow as you reasonably
  can (a specific environment beats a whole branch, a branch beats
  `pull_request`) since anyone who can trigger a matching run can read
  the vault.
- The `id-token: write` permission in both workflow files is required
  for the OIDC login step — don't remove it, and don't grant it more
  broadly than these two workflows need.
