# Encryption, Secrets, and API Security

## Encryption

| At rest | Data on disk encrypted (e.g. AES); keys in KMS or HSM. |
| In transit | TLS (TLS 1.2+) for client–server and service–service. |
| Key management | Rotate keys; separate encryption keys from data; use KMS for key storage and access control. |

**Interview:** “We encrypt at rest using the cloud KMS; we enforce TLS for all traffic; we rotate keys on a schedule and never log plaintext secrets.”

---

## Secrets Management

- **Don’t:** Hardcode in code, commit to git, or put in plaintext config.
- **Do:** Store in a secrets store (e.g. HashiCorp Vault, AWS Secrets Manager); inject at runtime (env, sidecar, or short-lived tokens).
- **Rotation:** Automate rotation; apps fetch fresh secret on startup or via webhook.

---

## API Security

- **HTTPS only** — no plain HTTP.
- **Rate limiting** — per user/key/IP to prevent abuse (see API Gateway).
- **Input validation** — validate and sanitize; prevent injection (SQL, command, XSS).
- **Principle of least privilege** — tokens and roles with minimal scope needed.
- **Audit logging** — log auth and sensitive actions for forensics.
