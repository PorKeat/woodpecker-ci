# 06 — User, Team & Admin Management

This guide explains how authentication, team organization access, and administrator rights work in Woodpecker CI.

---

## 1. Quick Summary of Settings

Woodpecker uses three core environment variables to govern user access:

| Setting | Production Value | What It Does |
| :--- | :--- | :--- |
| **`WOODPECKER_OPEN`** | `"false"` | **Closes public registration.** Random GitHub users cannot sign in. |
| **`WOODPECKER_ORGS`** | `unity-workspace-org` | **Team Whitelist:** Only members of the `unity-workspace-org` GitHub organization can log in. |
| **`WOODPECKER_ADMIN`** | `PorKeat` | **Superadmin Privileges:** Grants full administrative control over servers, runners, and global secrets. |

---

## 2. Who Can Log In?

Because `WOODPECKER_ORGS` is set to `unity-workspace-org`, all 8 active team members can sign in with their own GitHub accounts:
* `ChannimEY`
* `chaovanthoung168`
* `ericva01`
* `Leaphea-Lim`
* `Panharoth06`
* `PorKeat`
* `tochratana`
* `Vuthy-Tourn`

> [!TIP]
> Whenever you add or remove someone from the `unity-workspace-org` GitHub organization, their Woodpecker access is automatically granted or revoked.

---

## 3. How to Add a New Admin

There are **two ways** to grant admin rights:

### Method A: Web UI (Instant — Recommended)
As an existing admin (`PorKeat`):
1. Log in to [https://woodpecker.unity-workspace.com/](https://woodpecker.unity-workspace.com/).
2. Click your avatar (top-right) $\rightarrow$ **Administration** (or **Settings** $\rightarrow$ **Users**).
3. Find your teammate's username in the list.
4. Toggle the **Admin** switch to **ON** and click Save.
*Takes effect immediately with zero downtime.*

---

### Method B: Via Helm Values / Config (`WOODPECKER_ADMIN`)
Add multiple GitHub usernames separated by commas:

```yaml
WOODPECKER_ADMIN: PorKeat,ericva01,chaovanthoung168
```

Apply via kubectl:
```bash
ssh Node1 "kubectl set env statefulset/woodpecker-server -n woodpecker WOODPECKER_ADMIN='PorKeat,username2' -c server"
```

---

## 4. Admin vs. Member Permissions

| Feature | Regular Team Member | Admin |
| :--- | :---: | :---: |
| Log in with GitHub | ✅ | ✅ |
| View organization repositories | ✅ | ✅ |
| Trigger and view pipeline builds | ✅ | ✅ |
| View build logs | ✅ | ✅ |
| Manage repo-level secrets | ✅ (if repo admin on GitHub) | ✅ |
| Manage global Woodpecker settings | ❌ | ✅ |
| Manage Woodpecker agents/runners | ❌ | ✅ |
| Promote/demote other users | ❌ | ✅ |
