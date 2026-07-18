# monet-connectors

The community connector catalog for **Code Monet** (the desktop app at
[iaa2005/monet](https://github.com/iaa2005/monet)). The app's **Settings →
Connectors → Store** section reads this repository directly — merging a PR
here ships a connector to every user, no app update involved.

## How it works

A connector here is a **manifest — data, never code**. The app validates it
and builds the integration from its own built-in protocol engines
(IMAP/SMTP, WebDAV, CalDAV/CardDAV, local MCP). Installing a manifest cannot
execute JavaScript from this repo.

```
index.json                     ← the catalog the app lists
connectors/<id>/manifest.json  ← the full manifest
connectors/<id>/icon.svg       ← optional (inline SVG, viewBox, no scripts)
```

The app fetches `raw.githubusercontent.com/iaa2005/monet-connectors/main/…`.

## Adding a connector (PR checklist)

1. **Pick a stable id** — kebab-case, `[a-z0-9-]`, 2–41 chars. It can never
   change once shipped (user accounts reference it), and it must not collide
   with a built-in service id (`gmail`, `google-calendar`, `google-contacts`,
   `google-drive`, `yandex-mail`, `yandex-disk`, `yandex-calendar`,
   `yandex-contacts`, `telegram`, `telegram-bot`, `github`, `notion`,
   `slack`, `linear`, `sentry`).
2. **Create `connectors/<id>/manifest.json`** following `schema.json`
   (schema version `1`). See `connectors/mailru/` for a worked example.
3. **Verify every endpoint against the real server before submitting.**
   Never copy hosts from a blog post. Prove them:
   - mail: `openssl s_client -connect imap.example.com:993` (TLS must
     succeed and the certificate must belong to the provider);
   - webdav/caldav/carddav: `curl -i -X PROPFIND https://… -u probe:probe`
     — a `401` with a `WWW-Authenticate` header proves the protocol lives
     there (note: a `Basic realm` challenge alone does NOT prove that a
     password will be accepted — some providers advertise it and refuse);
   - mcp: `npm view <package> name version` — the package must exist, be the
     provider's official server, and run with `npx`/`uvx`.
   Put the verification commands and their output in the PR description.
4. **Add one entry to `index.json`** (id, name, company, description,
   version, capabilities).
5. **Write honest copy.** `note` and `setupSteps` must state the sharp
   edges: app-password activation delays, "share pages with the integration
   first", scopes to enable, and so on. The error hints (`authHint`) are
   what a user sees on a 401 — make them name what is actually checkable.

## What a manifest can and cannot do

| can | cannot |
|---|---|
| IMAP/SMTP mail (TLS only) | plaintext ports (`secure` must be `true`) |
| WebDAV files, CalDAV calendar, CardDAV contacts (https only) | http endpoints, URLs with embedded credentials |
| a local MCP server via `npx` / `uvx` | any other command — no `bash`, no binaries, no scripts |
| `password` / `token` auth forms | OAuth or phone-code sign-in flows (those need app code — contribute to the app instead) |

## Security model (why these rules exist)

- The user's credential goes to the hosts the manifest names. That is the
  attack surface: a malicious manifest is a **phishing form**. Mitigations:
  https/TLS-only, no credential-bearing URLs, and the app shows the user
  every endpoint before install — but the review here is the real gate.
  **Reviewers: check the hosts belong to the named provider.**
- An MCP entry IS code execution (a package runs on the user's machine).
  That's why `command` is allowlisted to package runners and reviewers must
  confirm the package is the provider's official server.
- `version` bumps ship updates; keep them semver-ish and describe changes in
  the PR.

## Manifest reference

See [`schema.json`](schema.json) for the machine-checkable contract. Field
notes beyond the schema:

- `name` — compact CompanyProduct style (`MailruMail`), used by the model
  and routines; the app derives the spaced display name from `company`.
- `company` — groups services in Settings (a company with 2+ services gets
  its own header). `""` for none.
- `auth.fields[].key` — `"username"` goes to the account's login; any other
  key lands in the encrypted secret store.
- `capabilities.mail.authHint` — appended to authentication failures; write
  the sentence that actually unblocks the user.
- `promptHint` — one sentence of guidance injected into the agent's tool
  prompt when this connector is connected.

Permissions are not declared here: every action gets the app's standard
matrix (reads allow, writes ask, destructive ask) and the user tunes it in
Settings.
