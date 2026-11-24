# Entra ID Project: SAML 2.0 SSO, OIDC Authentication & Posture Audit (INC-4022)

## Objective
Configure and validate enterprise Single Sign-On across **SAML 2.0** (`Crown-SAML` via `samltool.io`) and **OpenID Connect (OIDC)** (`Crown-OIDC` via `jwt.ms`) in Microsoft Entra ID, and pair portal testing with a read-only **Microsoft Graph PowerShell** posture audit to inspect application assignment requirements, reply/redirect URIs, and user MFA registration readiness.

## Environments & Prerequisites
- **Identity Provider (IdP):** Microsoft Entra ID Lab Tenant (`<tenant-id>`)
- **Test User Account:** Standard lab test user (`<test-user-upn>`, Object ID `<user-object-id>`)
- **SSO & Token Test Endpoints:** `https://samltool.io` (SAML 2.0 SP / ACS) and `https://jwt.ms` (OIDC Redirect URI / JWT decoder)
- **Automation Runtime:** PowerShell with the `Microsoft.Graph` SDK run from a dedicated lab workspace directory (`./IAM-lab`)
- **Required Microsoft Graph Permissions (Read-Only / Least Privilege):**
  - `Application.Read.All` (inspect service principals, app registrations, SSO protocols, and reply/redirect URLs)
  - `UserAuthenticationMethod.Read.All` (inspect user MFA registration and capability status)
  - `AuditLog.Read.All` (review directory and sign-in/audit context)
  - `Directory.Read.All` (read tenant directory objects)

## Tools & Controls Used
- **Microsoft Entra Enterprise Applications (SAML 2.0):** Configured Identifier/Entity ID and Reply/ACS URL (`https://samltool.io`), enforced explicit user assignment (`AssignmentReq = True`), and issued signed XML assertions (`<Issuer>`, `<Audience>`, `<NameID>`, `<AttributeStatement>`, time conditions).
- **Microsoft Entra App Registrations (OIDC / OAuth 2.0):** Registered `Crown-OIDC` (`aud` / Client ID `<client-id>`), enforced redirect URI allowlisting (`https://jwt.ms`), requested `openid profile email` scopes via the implicit `id_token` test flow, granted Global Admin consent, and inspected decoded JWT header (`alg: RS256`, `kid`) and payload claims (`iss`, `aud`, `sub`, `oid`, `preferred_username`, `exp`).
- **Microsoft Graph PowerShell:** Queried service principals, app registrations, and user authentication registration details (`Get-MgServicePrincipal`, `Get-MgApplication`, `Get-MgReportAuthenticationMethodUserRegistrationDetail`) and exported CSV reports to the lab workspace folder.

## Protocol & Audit Specification
- **SAML 2.0 Application:** `Crown-SAML` -> `https://samltool.io` (`AssignmentReq = True`, signed XML assertion for the assigned lab test user)
- **OIDC Application:** `Crown-OIDC` (Client ID `<client-id>`) -> `https://jwt.ms` (`AssignmentReq = False`, scopes `openid profile email`, admin consent granted)
- **Posture Audit Scope:** SSO protocol & assignment requirement check plus user MFA registration/capability readiness (`IsMfaRegistered`, `IsMfaCapable`, `softwareOneTimePasscode`) across lab accounts

## Methodology
- **Configure & Validate SAML 2.0 SSO (`Crown-SAML`):** Set Entity ID and Reply/ACS URL to `https://samltool.io`, enabled explicit user assignment (`AssignmentReq = True`), assigned the lab test user, and verified the signed SAML assertion (`Status: Successful`, valid signature, `<Issuer>`, `<Audience>`, `<NameID>`, `<AttributeStatement>`, and `NotBefore` / `NotOnOrAfter` window).
- **Configure & Validate OIDC Authentication (`Crown-OIDC`):** Registered `Crown-OIDC` with `https://jwt.ms` as the allowlisted redirect URI. Used `openid profile email` with `response_type=id_token` (implicit flow) strictly as a browser-based lab shortcut to inspect the `id_token` on `jwt.ms` without building a backend service—recognizing that production web and single-page applications should use **Authorization Code Flow with PKCE**. Resolved the initial user-consent block via Global Admin approval and verified the decoded JWT header (`RS256`, `kid`) and claims (`iss`, `aud`, `sub`, `oid`, `preferred_username`, `iat`, `nbf`, `exp`).
- **Run Automated Graph PowerShell Posture Audit:** Connected to Microsoft Graph with read-only scopes (using OAuth 2.0 delegated authorization under the hood), switched out of restricted system directories into a standard lab working folder to avoid `AccessDenied` on CSV exports, and reviewed application assignment settings and MFA registration readiness.

## Evidence

### 1. SAML Configuration (Entity ID, Reply URL![Uploading Screenshot 2026-11-23 200109.png…]()
)
![SAML Configuration](https://github.com/user-attachments/assets/4e9ba91a-635e-4522-8535-5eb0d01b4ced)

### 2. SAML Assertion (Issuer, Audience, NameID & Signature Validation)
![SAML Assertion](https://github.com/user-attachments/assets/65b2dba2-8221-40df-ade4-bad30203b0cb![Uploading Screenshot 2026-11-24 145652.png…]()
)

### 3. OIDC Configuration (App Registration & Redirect URI)
![OIDC Configuration](https://github.com/user-attachments/assets/d3d80e89-892c-46cf-bf1d-e0ee7a68d799)

### 4. Decoded JWT (Issuer, Audience, Subject & Expiry Claims)
![Decoded JWT](https://github.com/user-attachments/assets/1f1100e2-127a-4db6-9717-3060da28edba)

### 5. Graph Audit Output (SSO Configuration & MFA Registration Posture)
![Graph Audit Output](https://github.com/user-attachments/assets/d694ecdc-a85c-4947-ab80-a47d9c5958a4)

## Findings & Technical Distinctions
- **OAuth 2.0 vs. OIDC Responsibility:** OAuth 2.0 is an **authorization** framework (governing client registration, redirect URIs, delegated consent, and API access tokens such as `Connect-MgGraph` scopes), whereas **OIDC** adds an **authentication** layer on top of OAuth 2.0 using a signed `id_token` (JWT) to verify user identity.
- **Application Assignment Nuance (`AssignmentReq = False`):** `Crown-SAML` reported `AssignmentReq = True` (blocking unassigned users at the IdP), while `Crown-OIDC` reported `AssignmentReq = False`. Having `AssignmentReq = False` does not make the app anonymously accessible without authentication; rather, Entra ID does not require an explicit user or group assignment on the Enterprise Application before issuing a token, though tenant authentication and Conditional Access policies can still restrict access.
- **MFA Registration Readiness vs. Sign-In Enforcement:** Three lab accounts showed `IsMfaRegistered = True` (`softwareOneTimePasscode`), while one test account (`<unregistered-test-user>`) showed `IsMfaRegistered = False`. Importantly, `IsMfaRegistered` / `IsMfaCapable` measures whether users have enrolled and are capable of completing MFA—not whether MFA is actively enforced on every sign-in (which depends on Conditional Access or Security Defaults policy evaluation).

## Security & Governance Value
- **Standardizes Federated Authentication:** Centralizes identity verification in Entra ID across XML-based SAML enterprise apps and JSON/JWT-based OIDC applications without duplicating credentials.
- **Enforces Explicit Access & Consent Boundaries:** Combines explicit user assignment (`AssignmentReq = True` for restricted enterprise apps), redirect URI allowlisting, short-lived assertions/tokens, and admin consent to reduce spoofing, replay, and consent-phishing exposure.
- **Supports Repeatable Compliance & Risk Reviews:** Read-only Graph PowerShell queries surface application assignment posture and MFA enrollment gaps across the tenant.

## Lessons Learned
- Distinguishing between **authentication vs. authorization** (OIDC vs. OAuth 2.0), **lab implicit flow vs. production Auth Code + PKCE**, **assignment-required gating vs. unauthenticated access**, and **MFA registration vs. Conditional Access enforcement** is critical when translating technical audit outputs into accurate IAM governance findings.
- Running automation from a standard user workspace rather than a protected system directory avoids write permission errors and keeps CSV audit artifacts cleanly separated from system files.
