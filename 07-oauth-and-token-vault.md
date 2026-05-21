# 7. OAuth & Token Storage

Some flows require the user to authorize SlideMaker against an
external service — most commonly Google, for Google Drive (used by
the scheduled-decks Google Sheets data source) and Google Slides
export. This chapter describes how that consent is obtained, how the
resulting tokens are stored, and why the chosen scopes are narrower
than they could be.

## 7.1 Consent model: incremental, narrow scopes

A naive integration asks for every scope up front. This is a
verification hazard (broad scopes trigger Google's full security
review) and a UX hazard (a wall of consent toggles on first sign-in).

The model here is the opposite:

```
        Sign in
   (openid + email + profile)
           |
           v
       Authenticated
           |
   +-------+--------+
   |                |
   v                v
 "Pick a sheet"   "Export to Slides"
   click             click
   |                |
   |                |
   v                v
   Incremental    Incremental
   consent:       consent:
   drive.file     slides + drive.file
   |                |
   +-------+--------+
           |
           v
       Token store
```

Each feature triggers its own consent screen the first time it is
used. The scopes are the minimum needed for that feature. A user who
never uses Sheets or Slides never authorizes those scopes.

### The `inc:` state prefix

To distinguish "fresh sign-in" from "incremental consent for an
already signed-in user," the OAuth `state` parameter is prefixed with
`inc:` for incremental requests. The callback handler reads the
prefix:

- Plain state → new sign-in (create / merge user, issue session).
- `inc:` state → scope upgrade (attach new scopes to the existing
  user, keep the session).

```
   User                Frontend            Backend            Google
    |                     |                   |                  |
    | click "Pick sheet"  |                   |                  |
    | ------------------> |                   |                  |
    |                     | GET incremental   |                  |
    |                     | ----------------> |                  |
    |                     |                   | state = "inc:" + |
    |                     |                   |          nonce   |
    |                     | <-- redirect ---- |                  |
    |                     | ------------- consent (drive.file) - |
    |                     |                   |                  |
    |                     | <----------- redirect with code+st - |
    |                     | GET callback      |                  |
    |                     | ----------------> |                  |
    |                     |                   | detect "inc:"    |
    |                     |                   | attach scopes    |
    |                     |                   | exchange code -> |
    |                     |                   |                  |
    |                     |                   | save ciphertext  |
    |                     | <-- redirect back |                  |
```

## 7.2 Token storage at rest

Refresh tokens are long-lived secrets. If the database is exfiltrated,
the attacker can act as every user against every authorized provider
for as long as the tokens remain valid. To make that one-step
compromise into a two-step compromise, refresh tokens are encrypted at
rest with a symmetric key (Fernet, which is AES-128-CBC + HMAC) held
only by the application process.

```
    plaintext
   refresh_token
        |
        v
   +-----------+      +--------------------+
   |  Fernet   | <--- |  Application-only  |
   |  encrypt  |      |       key          |
   +-----+-----+      +--------------------+
         |
         v
    ciphertext
         |
         v
   +--------------+
   |   MySQL      |
   | oauth_token  |
   |  .ciphertext |
   +-----+--------+
         |
         | on use
         v
   +-----------+      +--------------------+
   |  Fernet   | <--- |  Application key   |
   |  decrypt  |      +--------------------+
   +-----+-----+
         |
         v
    plaintext
   refresh_token
         |
         v
   active access token
   via provider exchange
```

Properties:

- The encryption key is loaded into the application process at start
  time from a source the database does not have.
- A leaked database backup alone does not yield usable tokens.
- The key can be rotated by decrypting with the old key, re-encrypting
  with the new, and writing back.

### What is *not* encrypted

- Access tokens (short-lived) are never stored. They are fetched on
  demand by exchanging the refresh token, used immediately, and
  discarded.
- Metadata (`provider`, `scopes`, `granted_at`) is plaintext. It
  contains no secrets and is needed for routing.

## 7.3 Why `drive.file` instead of `spreadsheets.readonly`

The Google Sheets integration originally requested
`spreadsheets.readonly`. That is a **sensitive** scope under Google's
OAuth verification policy: it grants read access to *every*
spreadsheet in the user's Drive. To ship with that scope requires
going through Google's full security review.

The replacement is `drive.file`, a **non-sensitive** scope that grants
access only to files the user explicitly picks via the Google Picker
UI. The user picks; the application gets a file id; the application
reads that one file on subsequent runs.

```
   spreadsheets.readonly (sensitive)
   ---------------------------------
     consent: read ALL sheets
            |
            v
     app lists all of user's sheets
            |
            v
     user picks from app's list


   drive.file (non-sensitive)
   --------------------------
     consent: per-file access
            |
            v
     user pastes URL or picks via Google Picker
            |
            v
     app receives a single file id
     (and only that file's access)
```

### The Picker quirk

`drive.file` can only *list* files that have been previously granted
to the application. A naive Picker integration shows an empty list to
a new user. The workaround is to use the Picker's `setFileIds` API to
pre-navigate to a specific file that the user provides by URL paste.
The Picker treats this as a "grant access to this file"
confirmation; the file id then flows to the application.

More involved than a single "browse all sheets" picker, but it keeps
the integration in the non-sensitive band.

## 7.4 Token lifecycle

```
            (OAuth callback)
                 |
                 v
       +------------------+
       |     Granted      |
       +--------+---------+
                |
            first use
                |
                v
       +------------------+      +------------------+
       |      Active      | ---> |    Refreshed     |
       +--------+---------+      +--------+---------+
                ^                          |
                +--------------------------+
                |
                |  revoked / 400
                v
       +------------------+
       |     Expired      |
       +------------------+
                |
                v
              (end)
```

The Active <-> Refreshed loop is the common case. Expired (revoked by
the user, or marked invalid by the provider) terminates the token
and triggers a re-consent the next time the user invokes a feature
that needs the scope.

## 7.5 Session vs identity

A user identity is decoupled from the session token the browser
holds. One user can have multiple active sessions; one session
belongs to exactly one user. Sessions are server-side records keyed
by an opaque cookie. There is no JWT; the choice keeps revocation
simple (delete the row) and keeps the server-side state
authoritative.

```
   +---------------+      +-----------------+      +---------+
   |  session_id   | ---> | Session record  | ---> |  User   |
   |   (cookie)    |      |     (DB)        |      | record  |
   +---------------+      +-----------------+      +----+----+
                                                        |
                                                        v
                                                  +-------------+
                                                  |  OAuth      |
                                                  |  tokens     |
                                                  |  (encrypted)|
                                                  +-------------+
```

## 7.6 Defenses

A short list of practices stacked above the encryption:

- **Scope minimization.** Every scope is justified per feature.
- **Per-feature consent.** Users see what each feature needs at the
  moment they use it.
- **No long-lived access tokens at rest.**
- **Key separation.** The encryption key lives in a source the
  database does not have.
- **Auditable callback.** Every consent and every token issuance is
  logged (without ciphertext) with user id, provider, scopes, and
  timestamp.

## 7.7 Connections

- The Google Sheets / Drive flow is used by the scheduled-decks data
  sources in [chapter 8](08-scheduled-decks.md).
- The scope-narrowing decision is in
  [ADR-004](decisions/ADR-004-drive-file-and-picker.md).
- The token table sits in the broader storage model in
  [chapter 6](06-storage-model.md).
