# Authentication, Authorization, and OAuth

## Authentication (AuthN)

**Question:** Who are you? Verify identity (e.g. password, token, certificate).

| Method | Use case |
|--------|----------|
| **API key** | Service-to-service; simple; key in header or query. |
| **JWT (JSON Web Token)** | Stateless; signed; contains claims (user_id, scope, exp). Validate signature and exp; no DB lookup per request. |
| **Session cookie** | Browser; server stores session; cookie holds session ID. |
| **OAuth 2.0 / OIDC** | Delegated access; user logs in at IdP; client gets token to call APIs on user’s behalf. |

**Interview:** “We use JWTs for API auth so services can validate without a central session store; we keep TTL short and use refresh tokens for long-lived sessions.”

---

## Authorization (AuthZ)

**Question:** What can you do? Check permissions after identity is known.

- **RBAC (Role-Based):** User has roles; roles have permissions. “User is admin → can delete.”
- **ABAC (Attribute-Based):** Policies on attributes (user, resource, action, context). Fine-grained but complex.
- **Scope in token:** OAuth scope (e.g. read:users) checked at API; simple for APIs.

---

## OAuth 2.0 (High Level)

**Purpose:** Let a user grant a **client app** limited access to their resources (e.g. “read profile”) without giving the app their password.

- **Roles:** Resource owner (user), client (app), authorization server (issues tokens), resource server (API).
- **Flow (e.g. authorization code):** User redirected to auth server → logs in → consent → redirect back with code → client exchanges code for access token (and optionally refresh token).
- **Access token:** Sent to resource server (e.g. in Authorization header); resource server validates (signature or introspect at auth server).
- **Refresh token:** Used to get new access token when it expires; stored securely; can be revoked.

**Interview:** “We use OAuth 2.0 with authorization code flow for third-party apps; we issue short-lived access tokens and refresh tokens; API validates token and checks scope.”
