# JWT

A JSON Web Token is a self-contained, signed credential — a base64-encoded header, payload (claims: user id, expiry, scopes), and signature, joined by dots (`xxxxx.yyyyy.zzzzz`). Its defining feature is that a server can *verify* it (check the signature) without a database lookup, which is what makes JWTs popular for stateless API authentication: the token itself carries everything needed to trust it, until it expires.

## Not used in the nested app

This app makes no authenticated requests at all — see [[Authentication]] for why `GithubImportService` only ever calls GitHub's API unauthenticated, for public repos. There's no login flow, no session, and therefore no token of any kind (JWT or otherwise) anywhere in this codebase.

## Where a JWT would fit if this app added accounts

If the nested app ever added user accounts — say, to sync your folder tree across devices via a backend — a typical flow would be: the user logs in, the backend returns a JWT, the app stores it (via `flutter_secure_storage`, per [[Authentication]] and [[Encryption]]), and every subsequent API call attaches it:

```dart
final response = await http.get(
  Uri.parse('https://api.example.com/sync'),
  headers: {'Authorization': 'Bearer $jwt'},
);
```

The two things worth understanding conceptually, since there's no app code to point to: **expiry** — a JWT typically carries an `exp` claim, and a 401 response usually means it's expired and needs refreshing (often via a separate, longer-lived refresh token) rather than being wrong from the start; and **it's not encrypted, only signed** — anyone can base64-decode a JWT's payload and read its claims (try it on jwt.io), so a JWT should never be trusted to keep its contents secret, only to prove they haven't been tampered with. That second point is exactly why the token itself still needs secure client-side *storage* (protecting it from being copied/misused), even though the token's own payload isn't a secret you're hiding from its holder.

## Key takeaways

- JWT = signed, self-verifying credential — a server trusts it via signature check, not a session lookup, which is what makes it "stateless."
- The payload is readable by anyone with the token (base64, not encryption) — never put a genuine secret inside a JWT's claims.
- Expiry (`exp`) plus a refresh-token flow is the normal pattern for keeping a user logged in without one token being valid forever — relevant the moment this app (or any app) adds a real backend and login.
- This app has neither a backend nor login today — see [[Authentication]] for the one place (private GitHub repo support) where any credential handling would first become relevant, and it would likely be a personal access token, not a JWT the app itself issues.
