# 02 — GitHub OAuth App Setup

Woodpecker CI integrates with GitHub using an **OAuth 2.0 Application**. This allows your team to log in using their GitHub credentials and enables Woodpecker to clone repositories and receive webhook events.

---

## 1. Why GitHub Enforces Manual OAuth App Creation

GitHub strictly prohibits registering OAuth applications via REST API or CLI scripts for security reasons. An OAuth app can **only** be registered by an authenticated user through GitHub's Developer Settings UI.

---

## 2. Step-by-Step Registration (Takes 30 Seconds)

1. Open this direct registration link:
   👉 **[https://github.com/settings/applications/new](https://github.com/settings/applications/new)**
   *(Or under your organization: `https://github.com/organizations/unity-workspace-org/settings/applications/new`)*

2. Fill out the application details exactly:

   | Field | Value | Explanation |
   | :--- | :--- | :--- |
   | **Application name** | `Woodpecker CI` | Display name shown on login |
   | **Homepage URL** | `https://woodpecker.unity-workspace.com` | Base address of your Woodpecker dashboard |
   | **Application description** | `Internal Kubernetes CI/CD Engine` | Optional description |
   | **Authorization callback URL** | `https://woodpecker.unity-workspace.com/authorize` | **Critical:** GitHub redirects here after successful login *(No trailing slash!)* |

3. Click **Register application**.

4. Copy your **Client ID** (e.g. `Ov23lia9YFRFXGSycBne`).

5. Under **Client secrets**, click **Generate a new client secret** and copy it immediately.

---

## 3. Storing Credentials in HashiCorp Vault

Save them into HashiCorp Vault on `10.1.18.8`:

```bash
# Run on Vault VM (10.1.18.8) or from any machine with Vault CLI:
vault kv patch secret/woodpecker \
  github_client_id="<PASTE_YOUR_CLIENT_ID>" \
  github_client_secret="<PASTE_YOUR_CLIENT_SECRET>"
```

Verify that all secrets are saved:
```bash
vault kv get secret/woodpecker
```

---

## 4. Webhook URL

For each repository enabled in Woodpecker, GitHub will send events to:
```text
https://woodpecker.unity-workspace.com/api/hook?access_token=...
```
Woodpecker configures this automatically when you toggle a repository on in the dashboard.
