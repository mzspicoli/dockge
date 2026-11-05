# Passkey / OIDC login in front of Dockge (oauth2-proxy)

Dockge has no native OIDC. It does have the **Disable Auth** setting, meant for "using third-party authentication in front of Dockge".
This guide puts [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/) in front of Dockge so you sign in with any OIDC provider
(tested end to end with [Pocket ID](https://pocket-id.org), passkey-only, including Dockge's WebSocket; **not tested** with Authelia, Authentik or Keycloak, though any standards-compliant OIDC provider should work).

> **Security model.** With Disable Auth on, *anyone who reaches Dockge is admin*, and Dockge mounts `docker.sock` (= root on the host).
> The only safe setup is one where the proxy is the **only** network path to Dockge. Never publish port 5001 with Disable Auth on.

## 1. Create an OIDC client in your provider
- Redirect/callback URL: `https://dockge.example.com/oauth2/callback` (or `http://<host>:5002/oauth2/callback` on a private network)
- PKCE on (S256), confidential client. Note the client ID and secret (store the secret in a `.env` with mode 600, not in git).
- Restrict who may use the client in the provider too (e.g. a single user/group), not only in the e-mail allow-list below.

## 2. Compose (Dockge has NO published ports)
```yaml
services:
  dockge:
    image: louislam/dockge:1
    networks: [dockge-net]            # no "ports:" here
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
      - /opt/stacks:/opt/stacks
    environment:
      DOCKGE_STACKS_DIR: /opt/stacks

  dockge-auth:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.15.5   # pin by digest in production
    networks: [dockge-net]
    ports: ["127.0.0.1:5002:4180"]    # or your reverse proxy / private-network IP
    env_file: ./dockge-auth.env       # OAUTH2_PROXY_CLIENT_ID / CLIENT_SECRET / COOKIE_SECRET (chmod 600)
                                      # cookie secret: python3 -c 'import os,base64;print(base64.urlsafe_b64encode(os.urandom(32)).decode())'
    environment:
      OAUTH2_PROXY_PROVIDER: oidc
      OAUTH2_PROXY_OIDC_ISSUER_URL: https://id.example.com
      OAUTH2_PROXY_UPSTREAMS: http://dockge:5001
      OAUTH2_PROXY_REDIRECT_URL: https://dockge.example.com/oauth2/callback
      OAUTH2_PROXY_SCOPE: "openid profile email"
      OAUTH2_PROXY_CODE_CHALLENGE_METHOD: S256
      OAUTH2_PROXY_EMAIL_DOMAINS: "*"
      OAUTH2_PROXY_AUTHENTICATED_EMAILS_FILE: /etc/oauth2-proxy/emails.txt   # allow-list: one e-mail per line
      OAUTH2_PROXY_SKIP_PROVIDER_BUTTON: "true"
      OAUTH2_PROXY_COOKIE_SECURE: "true"   # false only on plain HTTP inside a private network
      OAUTH2_PROXY_COOKIE_EXPIRE: 8h
    volumes: ["./allowed-emails.txt:/etc/oauth2-proxy/emails.txt:ro"]
    read_only: true

networks:
  dockge-net: {}
```
Keep **only** `dockge` and `dockge-auth` on `dockge-net`.

## 3. Turn on Disable Auth
Create the Dockge user first (it is required to exist), then Settings → Security → *Disable Auth* (needs the current password).
Verify from outside: an unauthenticated request to `/` **and** `/socket.io/` must answer `302` to the provider, and port 5001 must refuse connections.

## Notes
- oauth2-proxy rejects unverified e-mails unless `OAUTH2_PROXY_INSECURE_OIDC_ALLOW_UNVERIFIED_EMAIL=true`; prefer verifying the e-mail in the provider.
- If the proxy or IdP is down, Dockge stays isolated; containers keep running. Re-enable the built-in login by resetting `disableAuth` in `data/dockge.db`.
- `Disable Auth` logs everyone in as the first Dockge user. Do not add other containers to `dockge-net`, and do not put another route to port 5001 (published port, second proxy, host networking).
- Sessions last as long as `COOKIE_EXPIRE`; revoke access by removing the e-mail from the allow-list and restarting `dockge-auth`.
