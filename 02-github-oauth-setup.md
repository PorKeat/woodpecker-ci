# 02 — GitHub OAuth App Setup

Woodpecker CI integrates with GitHub using an **OAuth 2.0 Application**. This allows you and your team to log in using your GitHub credentials and enables Woodpecker to clone your repositories and receive push/PR webhook events.

---

## 1. Why GitHub Enforces Manual OAuth App Creation

GitHub strictly prohibits registering OAuth applications via REST API or CLI scripts. GitHub enforces that an OAuth app can **only** be created by an authenticated human user clicking through GitHub's Developer Settings UI. This design protects your account against rogue scripts creating unauthorized OAuth apps that could impersonate services.

---

## 2. Step-by-Step Registration (Takes 30 Seconds)

1. Open this direct registration link:
   👉 **[https://github.com/settings/applications/new](https://github.com/settings/applications/new)**

2. Fill out the application details exactly as follows:

   | Field | Value | Explanation |
   | :--- | :--- | :--- |
   | **Application name** | `Woodpecker CI` | Display name shown on login |
   | **Homepage URL** | `https://10.1.16.200/woodpecker` | Base address of your Woodpecker dashboard |
   | **Application description** | `Internal Kubernetes CI/CD Engine` | Optional description |
   | **Authorization callback URL** | `https://10.1.16.200/woodpecker/authorize` | **Critical:** GitHub redirects here after successful login |

3. Click the green **Register application** button.

4. On the next page, you will see your **Client ID** (e.g. `Iv1.9f8a7b6c5d4e3f2a` or `Ov23li...`). Copy it.

5. Under **Client secrets**, click **Generate a new client secret**. Copy the generated secret immediately (GitHub only displays it once).

---

## 3. Storing the Credentials in HashiCorp Vault

Once you have your `Client ID` and `Client Secret`, save them directly into your Vault instance on `10.1.18.8`:

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

Output:
```
====== Metadata ======
...
====== Data ======
Key                   Value
---                   -----
access                vpn-only
admin_user            PorKeat
created_at            ...
github_client_id      Iv1.xxxxxxxxxxxx
github_client_secret  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
rpc_secret            c9ecc71b30612bfc502165628a06d9d7612663536d9662ab4e0b2dc839356f43
url                   https://10.1.16.200/woodpecker
```

---

## 4. Handling Webhooks with a Private IP / VPN Setup

Since `10.1.16.200` is on an internal private network:

### Strategy 1: Public Webhook Path with HMAC Secret (Recommended for Automatic Triggers)
If you configure an external proxy or Cloudflare tunnel, expose **only** the webhook endpoint (`/woodpecker/api/hook`) to GitHub's webhook IP ranges. Every webhook request sent by GitHub contains an HMAC SHA-256 signature in the `X-Hub-Signature-256` header, which Woodpecker verifies before triggering any pipeline.

### Strategy 2: Self-Hosted Runner / GitHub Action Forwarder
Add a 5-line GitHub Action in your repo that triggers Woodpecker via curl over your VPN tunnel when changes land on `main`.

### Strategy 3: Manual / UI Triggering
Developers trigger builds directly inside the Woodpecker web UI while connected to the VPN.
