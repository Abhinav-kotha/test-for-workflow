I have a task, and I need your help. I was given this workflow-1(obm_obm-actions/.github/workflows
/build-deploy-to-gcp-api-testing.yml) and I was also given this workflow-2(obm_imd-customer-profile-api/.github/workflows
/caller-workflow-dev.yaml). Now my task is to replace the github secret with secret manager's secret(at around 239th line ) for this variable GH_TOKEN.(I have attached both workflow files)
To **replace the GitHub Secret (`GH_TOKEN`)** in your workflow with a **Secret Manager secret**, you can follow these steps, which assume you're using **Google Cloud Secret Manager**.

---

### ✅ Objective:

In `build-deploy-to-gcp-api-testing.yml`, `GH_TOKEN` is currently fetched via:

```yaml
GH_TOKEN: ${{ secrets.GH_TOKEN }}
```

You want to **replace this** with a token **fetched from GCP Secret Manager at runtime**, not GitHub Secrets.

---

### 🛠️ Step-by-step Changes:

#### 1. **Add a step to fetch GH\_TOKEN from Secret Manager**

You can use the `google-github-actions/get-secretmanager-secrets@v1` action to retrieve the token from GCP Secret Manager.

🔽 **Insert this step before any usage of `GH_TOKEN`**:

```yaml
- name: Fetch GitHub Token from Secret Manager
  id: fetch-gh-token
  uses: google-github-actions/get-secretmanager-secrets@v1
  with:
    secrets: |
      GH_TOKEN:projects/YOUR_PROJECT_ID/secrets/YOUR_SECRET_NAME
```

> 🔁 Replace `YOUR_PROJECT_ID` and `YOUR_SECRET_NAME` with the actual project ID and secret name in GCP Secret Manager.

This stores the value in `steps.fetch-gh-token.outputs.GH_TOKEN`.

---

#### 2. **Replace all usages of `secrets.GH_TOKEN`**

Go through the file and replace any line like:

```bash
-H "Authorization: token ${{ secrets.GH_TOKEN }}"
```

with:

```bash
-H "Authorization: token ${{ steps.fetch-gh-token.outputs.GH_TOKEN }}"
```

✅ You’ll need to update **each place** `secrets.GH_TOKEN` appears:

* Curl requests to GitHub API
* Git config for remote origin
* `gh` CLI commands (for PR creation and merging)
* Env variable setting like:

  ```yaml
  - name: "Set GitHub Token"
    run: echo "GH_TOKEN=${{ secrets.GH_TOKEN }}" >> $GITHUB_ENV
  ```

  should become:

  ```yaml
  - name: "Set GitHub Token"
    run: echo "GH_TOKEN=${{ steps.fetch-gh-token.outputs.GH_TOKEN }}" >> $GITHUB_ENV
  ```

---

### 🔐 Permissions

Make sure your GitHub Actions runner is authenticated with a GCP service account that has access to read the specified secret from Secret Manager.

That means the secret-accessing step should **come after this block**:

```yaml
- name: 'Authenticate to Google Cloud'
  uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets[env.SA_KEY] }}
```

---

### ✅ Summary of What You Need to Modify:

**In `build-deploy-to-gcp-api-testing.yml`:**

1. After GCP authentication, **insert** the "Fetch GitHub Token from Secret Manager" step.
2. Replace **every** `secrets.GH_TOKEN` usage with `steps.fetch-gh-token.outputs.GH_TOKEN`.

---

Would you like me to generate a **patch version of the updated YAML** for you to copy directly?
