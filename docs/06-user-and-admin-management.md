# 👤 User & Admin Management Guide

This guide explains how logins and admin permissions work in Woodpecker CI.

---

## 1. Quick Config Summary

In `manifests/woodpecker/03-helm-values.yaml`:

```yaml
WOODPECKER_OPEN: "false"             # 1. Blocks random strangers from signing up
WOODPECKER_ORGS: unity-workspace-org # 2. Allows all team members in this GitHub Org
WOODPECKER_ADMIN: PorKeat            # 3. Superadmin account
```

---

## 2. Who Can Log In?

Anyone who is a member of your GitHub organization **`unity-workspace-org`** can log in:
* `ChannimEY`
* `chaovanthoung168`
* `ericva01`
* `Leaphea-Lim`
* `Panharoth06`
* `PorKeat` (Admin)
* `tochratana`
* `Vuthy-Tourn`

> Anyone NOT in this GitHub organization will be blocked automatically.

---

## 3. How to Make a Teammate an Admin

### Option A: From the Web UI (Takes 5 seconds, No Restart)
1. Go to **[https://woodpecker.unity-workspace.com/](https://woodpecker.unity-workspace.com/)**.
2. Click your avatar (top-right) $\rightarrow$ **Administration** $\rightarrow$ **Users**.
3. Find your teammate's name.
4. Toggle **Admin** to **ON** and click Save.

### Option B: In the Configuration File
Add multiple usernames separated by a comma:
```yaml
WOODPECKER_ADMIN: PorKeat,ericva01,chaovanthoung168
```
Then update the server:
```bash
ssh Node1 "kubectl set env statefulset/woodpecker-server -n woodpecker WOODPECKER_ADMIN='PorKeat,ericva01' -c server"
```

---

## 4. What Can Members vs. Admins Do?

* **Regular Team Members**:
  * Log in with GitHub.
  * View team repositories and pipeline history.
  * Trigger manual builds.
  * View build logs.

* **Admins**:
  * Everything above.
  * Manage global secrets (like GHCR Docker tokens).
  * Manage Woodpecker agents and runners.
  * Promote other users to Admin.
