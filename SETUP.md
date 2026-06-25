# SAMY Data Products Decision Tool — Login Setup

The tool now opens on a **Sign in with Google** screen and only lets in
accounts on the **`@samy.com`** Google Workspace domain.

There are two layers, and you should do both:

1. **In-app Google Sign-In** (done — works on GitHub Pages today). You just
   need to add your OAuth Client ID.
2. **Cloudflare Access** (recommended hardening) for *true* enforcement, since
   anything running in the browser can be bypassed by a technical user.

---

## 1. Add your Google OAuth Client ID (required to turn on login)

1. Go to the Google Cloud Console → **APIs & Services → Credentials**:
   https://console.cloud.google.com/apis/credentials
2. Pick (or create) a project owned by the SAMY Workspace.
3. Configure the **OAuth consent screen**:
   - User type: **Internal** (this alone limits sign-in to @samy.com).
   - Fill in app name, support email, developer email.
4. **Create Credentials → OAuth client ID**:
   - Application type: **Web application**.
   - **Authorized JavaScript origins** — add every origin the page is served from, e.g.:
     - `https://lucasrios-prodmgt.github.io`
     - your custom domain if you add one (e.g. `https://tools.samy.com`)
     - `http://localhost:8000` (only if you test locally)
   - You can leave "Authorized redirect URIs" empty — Google Identity Services
     uses the origins above.
5. Copy the generated **Client ID** (looks like `1234567890-abc.apps.googleusercontent.com`).
6. Open `SAMY-Data-Products-Decision-Tool.html`, find this line near the bottom:

   ```js
   const GOOGLE_CLIENT_ID = "YOUR_GOOGLE_CLIENT_ID.apps.googleusercontent.com";
   ```

   Replace the placeholder with your real Client ID. Commit and push.

That's it — the login screen will then show the real Google button. Until the
Client ID is set, the page shows a yellow "not configured yet" note.

### How the domain restriction works in-app
- The button is initialized with `hd: "samy.com"` so Google steers users to
  their SAMY account.
- After sign-in, the code checks the ID token's `hd` claim **and** that the
  email ends in `@samy.com`, and that the email is verified. Anything else is
  rejected with an error message.
- A valid session is kept in `sessionStorage` for the tab so users aren't
  re-prompted on reload. A **Sign out** chip appears in the header.

> ⚠️ **Note on the "Internal" consent screen:** setting the consent screen to
> *Internal* means Google itself will refuse non-@samy.com accounts at the
> Google login step. That's stronger than the in-page check, but still relies
> on the client. For a hard boundary, add Cloudflare Access below.

---

## 2. Hard enforcement with Cloudflare Access (recommended)

Because GitHub Pages is static, the in-app check can be bypassed by a technical
user editing the page. Cloudflare Access enforces login **at the edge, before
the page is ever served** — no app code involved. Free for up to 50 users.

High-level steps:

1. **Add a custom domain on Cloudflare.** Put a domain/subdomain (e.g.
   `tools.samy.com`) on Cloudflare DNS, pointing (via CNAME) at your GitHub
   Pages site. Configure the custom domain in your repo's Pages settings too.
2. In the Cloudflare dashboard open **Zero Trust → Access → Applications →
   Add an application → Self-hosted.**
   - Application domain: `tools.samy.com`.
3. **Add a policy:**
   - Action: **Allow**
   - Include rule: **Emails ending in** → `@samy.com`
     (or **Identity provider = Google Workspace** scoped to the SAMY domain).
4. Under **Settings → Authentication**, add **Google** (or Google Workspace) as
   the login method.
5. Save. Now visiting `tools.samy.com` forces a Google login and rejects any
   non-`@samy.com` user before the decision tool loads.

With Access in front, the in-app login becomes a friendly second layer and the
domain rule is genuinely enforced.

---

## Quick local test

```bash
# from the repo folder
python3 -m http.server 8000
# open http://localhost:8000/SAMY-Data-Products-Decision-Tool.html
```

Add `http://localhost:8000` to the Authorized JavaScript origins first, or the
Google button will report an origin error.
