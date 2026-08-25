# Heal Grow Blossom Therapy — Infrastructure Setup

Runbook for standing up email (Google Workspace) and hosting (Netlify) for
**healgrowblossomtherapy.com**. Domain is registered at **Porkbun** — all DNS
records below get added in Porkbun's DNS panel.

## Current state (verified 2026-08-25)

Everything below is done and live. The checklist is kept as a runbook in case
any of it ever has to be redone (provider migration, DNS reset, etc.).

- **Mail:** Google Workspace. MX → `smtp.google.com`; SPF, DKIM (`google`
  selector, 2048-bit) and DMARC all pass. DMARC is at `p=quarantine` with no
  `rua=` — aggregate reports were dropped on purpose (see Phase 1, step 6).
- **Web:** Netlify site `shimmering-taffy-7f844c.netlify.app`, deployed from
  `github.com/Blossom-Therapy/blossom` (`main`). Primary domain is the apex;
  `www` and the `.netlify.app` subdomain both 301 to it. HTTPS via Let's Encrypt.
- **Pending:** move DMARC to `p=reject` after ~2026-09-08 if nothing has
  landed in spam.

**Porkbun DNS field convention:** the **Host** field is the subdomain *without*
the domain. Leave it **blank** (or `@`) for the apex/root. Enter just `www`,
`google._domainkey`, `_dmarc`, etc. for subdomains — Porkbun appends the domain
automatically.

Two values below are account-specific and must be copied out of the Google
Admin console — they're marked **⟨FROM GOOGLE⟩**.

---

## Phase 0 — Clear Porkbun's default records first

A freshly registered Porkbun domain ships with 7 auto-created records: a parking
page, default email forwarding, and leftover SSL-validation tokens. **None were
added by you, and all should be removed** before adding Google/Netlify records —
three of them would actively conflict with Google email.

- [x] `ALIAS @ → uixie.porkbun.com` — parking page. Remove (Netlify apex ALIAS replaces it).
- [x] `CNAME *.@ → uixie.porkbun.com` — wildcard to parking. Remove.
- [x] `MX @ → fwd1.porkbun.com` (prio 10) — Porkbun forwarding. **Remove** (conflicts with Google MX).
- [x] `MX @ → fwd2.porkbun.com` (prio 20) — Porkbun forwarding. **Remove** (same).
- [x] `TXT @ → v=spf1 include:_spf.porkbun.com ~all` — Porkbun SPF. **Remove** (only one SPF allowed).
- [x] `TXT _acme-challenge → …` (×2) — stale SSL validation tokens. Safe to remove.

> The two `MX` records and the `_spf.porkbun.com` `TXT` are the dangerous ones —
> leaving them breaks Google mail routing and SPF. The `_acme-challenge` records
> are harmless but serve no purpose once off Porkbun's parking page.

---

## Phase 1 — Google Workspace (email + full app suite)

Business Starter ($7/user/mo, annual) already includes the entire suite for
Marcia — Gmail, Calendar, Drive, Docs/Sheets/Slides, Meet, Chat — with 30 GB
storage. Upgrade to Business Standard later only if she needs 2 TB storage or
larger Meet recordings. One mailbox = one user = one fee.

### Steps
- [x] **1. Sign up** at workspace.google.com → Business Starter. Set the primary
      user to `marcia@healgrowblossomtherapy.com` (this is also the admin
      account).
- [x] **2. Verify domain ownership.** Google gives you a TXT value like
      `google-site-verification=abc123…`. Add it to Porkbun (see table), then
      click **Verify** in the Google setup wizard.
- [x] **3. Add the MX record** so mail routes to Gmail (see table).
- [x] **4. Add the SPF record** (see table). ⚠️ Only **one** `v=spf1` TXT record
      may exist on the domain — never create a second.
- [x] **5. Turn on DKIM.** In the Google **Admin console** →
      *Apps → Google Workspace → Gmail → Authenticate email* → select the domain
      → **Generate new record** (2048-bit; selector is `google`). Copy the value,
      add the TXT record in Porkbun (see table), wait a few minutes for it to
      propagate, then click **Start authentication** back in the console.
- [x] **6. Add the DMARC record** (see table). Ran at `p=none` with
      `rua=mailto:marcia@…` first; Google and Comcast reports showed every
      message passing SPF + DKIM, so on 2026-08-25 it was tightened to
      `p=quarantine` and `rua=` was dropped — the reports were inbox noise with
      no monitoring service reading them. Re-add `rua=` only if one is ever set
      up. Only Gmail sends as this domain (SimplePractice sends from its own
      domain, so it doesn't affect alignment). **Next:** after ~2 weeks with
      nothing landing in spam, edit the same record to `p=reject`. Only one
      `_dmarc` record may exist.
- [x] **7. Verify** with https://toolbox.googleapps.com/apps/checkmx/ — it
      confirms MX, SPF, DKIM, and DMARC in one shot.

### Google DNS records (add in Porkbun)

| # | Type  | Host              | Value / Answer                                        | Priority |
|---|-------|-------------------|-------------------------------------------------------|----------|
| 2 | TXT   | *(blank / @)*     | `⟨FROM GOOGLE⟩ google-site-verification=…`             | —        |
| 3 | MX    | *(blank / @)*     | `smtp.google.com`                                     | `1`      |
| 4 | TXT   | *(blank / @)*     | `v=spf1 include:_spf.google.com ~all`                | —        |
| 5 | TXT   | `google._domainkey` | `⟨FROM GOOGLE⟩ v=DKIM1; k=rsa; p=…`                 | —        |
| 6 | TXT   | `_dmarc`          | `v=DMARC1; p=quarantine` (→ `p=reject` after ~2026-09-08) | — |

---

## Phase 2 — Netlify (website hosting)

Independent of Phase 1 — the website records (ALIAS/A + www CNAME) and the email
records (MX/TXT) coexist without conflict. You can do this in parallel, and you
can stand up the whole pipeline with a placeholder page *before* the real
`index.html` exists.

### Steps
- [x] **1. Create a Netlify account** (free "Starter" tier, no card). Sign up
      **with GitHub** so deploys are git-based.
- [x] **2. Create a GitHub repo** (`Blossom-Therapy/blossom`) and push a placeholder
      `index.html` — even one line. (This `SETUP.md` can live in it too.)
- [x] **3. Connect the repo** in Netlify → it auto-deploys to a temporary
      `shimmering-taffy-7f844c.netlify.app` URL. Confirm the placeholder loads.
- [x] **4. Add the custom domain** in Netlify:
      *Site → Domain management → Add a domain* → `healgrowblossomtherapy.com`.
      Netlify will display the exact records to add (they'll match the table
      below).
- [x] **5. Add the DNS records** in Porkbun (see table).
- [x] **6. Pick the primary domain** in Netlify (apex vs `www`) — it auto-creates
      the redirect for the other one. Apex is primary here.
- [x] **7. HTTPS** provisions automatically (Let's Encrypt) once DNS resolves —
      usually minutes, up to a day. No action needed.
- [x] **8. Redirect the `.netlify.app` subdomain.** Netlify only auto-redirects
      `www` ↔ apex; the default subdomain keeps serving a duplicate of the site
      unless told otherwise. `_redirects` at the repo root handles it:
      `https://shimmering-taffy-7f844c.netlify.app/* https://healgrowblossomtherapy.com/:splat 301!`

### Netlify DNS records (add in Porkbun)

| Type   | Host          | Value / Answer                    | Notes                                      |
|--------|---------------|-----------------------------------|--------------------------------------------|
| ALIAS  | *(blank / @)* | `apex-loadbalancer.netlify.com`   | Preferred at apex; Porkbun supports ALIAS. |
| CNAME  | `www`         | `shimmering-taffy-7f844c.netlify.app` | The Netlify site name.                 |

> Apex fallback: if ALIAS ever gives you trouble, replace it with an **A** record
> at the apex pointing to `75.2.60.5`. Do **not** use a CNAME at the apex — it
> would break the MX/email records.

---

## Notes & gotchas
- **Propagation:** DNS changes can take up to ~24h globally, though usually far
  faster. If email or the site doesn't work immediately, give it time before
  debugging.
- **One SPF record only.** If a second `v=spf1` TXT ever appears, SPF breaks.
- **Don't mix MX styles.** The single `smtp.google.com` record replaces the old
  5-record `ASPMX.L.GOOGLE.COM` set — use one style, not both.
- **The domain is the durable asset.** Because you control DNS at Porkbun, you
  can change any provider behind it later with zero disruption to the address.

## Verification tools
- Email: https://toolbox.googleapps.com/apps/checkmx/
- Any record: https://mxtoolbox.com/ (SuperTool — check MX, SPF, DKIM, DMARC)
- DNS propagation: https://dnschecker.org/
