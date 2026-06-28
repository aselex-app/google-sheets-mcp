# Connecting Your Google Account

This is the step-by-step guide most new users are looking for: how to get the
server talking to **your** Google account so it can read and write your sheets.

There are **three phases**. Phase A is a one-time setup; after that you only ever
repeat Phase B if you switch accounts or revoke access.

```
A. Get an app credential   →   B. Connect your account   →   C. Wire it into your client
   (Google Cloud, ~5 min)       (OAuth login, ~30 sec)         (Claude Desktop / CLI)
```

---

## Phase A — Get a `credentials.json` (one-time, ~5 minutes)

A `credentials.json` is the *application* key that lets the server talk to Google
APIs. It is **not** your account — you log in with your account in Phase B.

1. Open the **Google Cloud Console** → <https://console.cloud.google.com>
2. Create a new project (or select an existing one).
3. **APIs & Services → Library** — enable both:
   - **Google Sheets API**
   - **Google Drive API**
4. **APIs & Services → OAuth consent screen**:
   - User type: **External**
   - Fill in the app name and your email.
   - Under **Test users**, add the Google account you'll connect (otherwise Google
     blocks the login while the app is unverified).
5. **APIs & Services → Credentials → Create credentials → OAuth client ID**:
   - Application type: **Desktop app**
   - Click **Create**, then **Download JSON**.
6. Rename the downloaded file to `credentials.json` and place it in the server's
   credentials directory:
   ```bash
   mkdir -p ~/.config/google-sheets-mcp
   mv ~/Downloads/client_secret_*.json ~/.config/google-sheets-mcp/credentials.json
   chmod 600 ~/.config/google-sheets-mcp/credentials.json
   ```

> 💡 You only do Phase A once. The same `credentials.json` works for any number of
> connections.

---

## Phase B — Connect your account (OAuth login, ~30 seconds)

Run the interactive setup wizard:

```bash
google-sheets-mcp --setup
```

What happens:

1. The wizard opens your browser.
2. **You pick your Google account and sign in** — this is the moment your account
   gets connected.
3. You grant the requested permissions (see the scopes below).
4. A `token.json` is saved to `~/.config/google-sheets-mcp/token.json`.

That's it — your account is connected. The token refreshes automatically, so you
won't need to log in again unless you revoke access or delete `token.json`.

### What permissions you're granting (least privilege)

| Scope | What it allows |
|-------|----------------|
| `openid`, `userinfo.email` | Identify which account is connected |
| `…/auth/spreadsheets` | **Read & write** the spreadsheets you have access to |
| `…/auth/drive.metadata.readonly` | List/search spreadsheets and read their metadata |

This server **never** requests the broad `…/auth/drive` scope. It cannot touch any
non-spreadsheet files in your Drive — the consent screen reflects exactly these
limited permissions.

---

## Phase C — Wire it into your MCP client

Add the server to your client's configuration, pointing at the project's `venv`
Python and your credentials directory.

**Claude Desktop** (`claude_desktop_config.json`) **or Claude Code** (`~/.claude.json`):

```json
{
  "mcpServers": {
    "google-sheets": {
      "command": "/path/to/google-sheets-mcp/venv/bin/python",
      "args": [
        "-m",
        "google_sheets.server",
        "--credentials-dir",
        "/Users/YOU/.config/google-sheets-mcp"
      ]
    }
  }
}
```

Then **restart the client**. The Google Sheets tools will appear and you can start
asking things like *"List all my spreadsheets."*

---

## Alternative connection methods

OAuth (above) is best for personal use. Two other methods are supported:

### 🤖 Service Account (best for automation / servers)

No interactive login. Instead you create a service account, download its JSON key,
and **share each spreadsheet with the service account's email** (it acts like a
robot user).

```bash
google-sheets-mcp --setup   # choose "Service Account" and point to the key file
```

### 🌐 Application Default Credentials (best if you already use GCP)

Reuse your existing `gcloud` login:

```bash
gcloud auth application-default login \
  --scopes="https://www.googleapis.com/auth/spreadsheets,https://www.googleapis.com/auth/drive.metadata.readonly"
```

---

## Troubleshooting

| Symptom | Fix |
|--------|-----|
| **"Access blocked: app not verified"** | Add your email under **OAuth consent screen → Test users** (Phase A, step 4). |
| **"Authentication failed" on startup** | Re-run `google-sheets-mcp --setup`; delete `~/.config/google-sheets-mcp/token.json` to force a fresh login. |
| **"Insufficient permissions"** | Make sure both Sheets API and Drive API are enabled, and that the connected account actually has access to the spreadsheet. |
| **Service account can't see a sheet** | Share the spreadsheet with the service account's email address. |
| **`ModuleNotFoundError`** | Check the client config points at the project `venv` Python and uses `-m google_sheets.server`. |

For more, see the main [README](../README.md) and [SECURITY.md](../SECURITY.md).
