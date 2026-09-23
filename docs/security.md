# Security Requirements

## Cloudflare Zero Trust — Mandatory

**The DE Framework portal MUST be deployed behind Cloudflare Zero Trust (Access).** This is not optional.

### Why

The portal exposes:
- Live KPI data for all Digital Employees
- Pending Level-2 decisions (awaiting human approval)
- Experiment history and workspace contents
- DE session logs

Without Zero Trust, anyone with the URL can read your operational data and approve decisions.

### What to Protect

| Surface | Protection required |
|---------|-------------------|
| Portal (`portal.your-domain.com`) | CF Zero Trust — email policy |
| CF Pages preview (`*.pages.dev`) | CF Zero Trust — same policy |
| Backend API (`de-api.your-domain.com`) | Bearer token (already enforced by de-backend) |
| DE workspaces (`/var/de-agents/`) | Filesystem — not publicly exposed |

### Setup (CF Dashboard — 5 minutes)

The CF API token does **not** have Access permissions. This must be done manually:

1. Go to **https://one.dash.cloudflare.com**
2. **Access → Applications → Add an application → Self-hosted**
3. Fill in:
   - Name: `DE Framework Portal`
   - Domain: `portal.your-domain.com`
   - Additional domain: `your-project.pages.dev`
4. **Policies → Add policy:**
   - Rule: Emails → add authorized email addresses
5. Save

After this, visiting either URL redirects to a Cloudflare login page. Only listed emails can proceed.

### Verify It's Working

```bash
# Should return a redirect to Cloudflare Access, not the portal
curl -s -o /dev/null -w "%{http_code}" https://portal.your-domain.com
# Expected: 302 (redirect to CF login) or 403 — NOT 200
```

If you get `200` without logging in, Zero Trust is not active.

### Backend API Security

The backend is separately protected by a Bearer token (`DE_API_TOKEN` in `/etc/de-framework.env`). Zero Trust is for the human-facing portal only — the backend API is accessed by DEs internally and does not go through CF Access.

```bash
# Valid request (with token)
curl -H "Authorization: Bearer ***" https://de-api.your-domain.com/de-list

# Invalid (no token) → 401
curl https://de-api.your-domain.com/de-list
```

### CF Access Token for API Calls (Optional)

If you want to call the portal URL programmatically from a script (e.g., for monitoring), generate a CF Access service token:

1. CF Dashboard → Access → Service Auth → Create Service Token
2. Use `CF-Access-Client-Id` and `CF-Access-Client-Secret` headers

For normal DE operations, this is not needed.
