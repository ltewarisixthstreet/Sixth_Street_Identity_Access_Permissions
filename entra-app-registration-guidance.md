# App Registration Guidance: Non-Prod & Prod Entra (Microsoft Entra ID)
### OAuth Integration Developer Guide

---

## What Is Microsoft Entra ID — and What Is an "App Registration"?

**Microsoft Entra ID** (formerly Azure Active Directory / Azure AD) is the identity and access management platform used across the organization. It is the system that handles *who you are*, *what you're allowed to do*, and *how applications authenticate users and services* — including all OAuth 2.0 and OpenID Connect (OIDC) flows.

When a developer wants to build an application that:
- Lets users **sign in with their corporate credentials** (SSO),
- Calls a **protected API** (e.g., Microsoft Graph, an internal API secured by Entra),
- Acts as a **service/daemon** that authenticates as itself (client credentials flow), or
- Participates in any **OAuth 2.0 / OIDC** integration,

…they must first create an **App Registration** inside Entra ID. Think of an App Registration as the *identity card* for your application. It tells Entra:

- "This application exists."
- "Here is what it is allowed to ask for (scopes/permissions)."
- "Here are the redirect URIs it is allowed to use."
- "Here is the secret or certificate it will use to prove its identity."

Without an App Registration, Entra has no knowledge of your application and will refuse all authentication requests from it.

---

## Our Two Entra Environments

Like most enterprise systems, we maintain **two separate Entra tenants** (environments):

| Environment | Purpose | Who Can Create App Registrations |
|---|---|---|
| **Non-Prod Entra** | Development, testing, and integration validation | Developers (self-service, with guardrails) |
| **Prod Entra** | Live, customer-facing, and business-critical workloads | Controlled — requires a formal promotion process |

These are **completely separate tenants** with separate tenant IDs, separate user directories, separate secrets, and separate permission grants. A client ID and secret from Non-Prod will **never work** in Prod, and vice versa.

---

## When Does "Creating an App in Prod Entra" Come Up?

As a developer, you will encounter the need for a Prod Entra App Registration in the following scenarios:

**1. Launching a new application or service to production**
Any application that needs to authenticate real users or call production APIs must have a corresponding App Registration in Prod Entra before it can go live.

**2. Enabling SSO for a production workload**
If your application will allow employees or external users to sign in using their organizational credentials in the production environment, a Prod App Registration is required to configure the trust relationship.

**3. Calling production-tier Microsoft APIs (e.g., Microsoft Graph)**
Accessing production data via Graph API — mailboxes, SharePoint, Teams, user profiles — requires a Prod App Registration with the appropriate permissions granted by an admin.

**4. Service-to-service authentication in production**
If a backend service or daemon needs to authenticate to another production API using the client credentials flow (no user involved), it needs its own Prod App Registration with a client secret or certificate.

**5. Integrating a third-party SaaS tool with the organization's identity**
When onboarding a vendor tool that needs to federate with our Entra tenant (e.g., a new SaaS platform that supports SAML or OIDC SSO), a Prod App Registration is created to establish that trust.

---

## Why We Cannot Allow Open Self-Service App Creation in Prod Entra

This is not bureaucracy for its own sake. Unrestricted App Registration creation in Prod Entra carries **real organizational risk**. Here is why access is controlled:

### 1. Permission Sprawl and Over-Privileged Apps
Entra permissions (OAuth scopes) can be extremely broad. A developer unfamiliar with the principle of least privilege might request `User.ReadWrite.All` when they only need `User.Read`. In Prod, that over-privileged app now has write access to every user account in the directory. Mistakes like this in Non-Prod are recoverable; in Prod, they are security incidents.

### 2. Unreviewed Redirect URIs Create Phishing Vectors
OAuth flows rely on redirect URIs to send tokens back to the application. A misconfigured or overly permissive redirect URI (e.g., a wildcard or an attacker-controlled domain) in a Prod App Registration can be exploited to redirect tokens to a malicious destination — a classic OAuth redirect attack.

### 3. Orphaned App Registrations
Developers move teams, projects get cancelled, and people leave. An App Registration created informally in Prod with no documented owner becomes an orphan — it may still hold active secrets, broad permissions, and service principal grants, but nobody knows what it does or whether it is still needed. Orphaned apps are a persistent audit and security finding.

### 4. Secret Management and Rotation
Client secrets created in Prod App Registrations must be stored securely, rotated on schedule, and tracked. Informally created apps often have secrets stored in ad-hoc locations (local `.env` files, personal vaults, Slack messages) with no rotation plan — a significant credential exposure risk.

### 5. Compliance and Audit Requirements
Our Prod Entra tenant is subject to compliance frameworks (e.g., SOC 2, ISO 27001, internal security policy). Every App Registration in Prod must be justifiable, documented, and attributable to an approved business need. Uncontrolled creation breaks the audit trail.

### 6. Admin Consent Cannot Be Undone Casually
Many API permissions in Entra require **admin consent** — a tenant-wide grant that applies to all users. Once an admin grants consent to an over-privileged app, revoking it requires deliberate action and may break the app. Getting this wrong in Prod has immediate, broad impact.

---

## Alternatives for Testing — Use Non-Prod Entra

**All development and integration testing must be done against Non-Prod Entra.** Here is how to work effectively within that environment:

### Getting Started in Non-Prod

1. **Request a Non-Prod App Registration** via the standard intake process (link to your team's intake form/ticket queue). Turnaround is typically same-day for standard configurations.
2. You will receive a **Client ID** (Application ID) and instructions for generating a **Client Secret** or uploading a certificate.
3. Configure your application to point at the **Non-Prod tenant's authority URL** (e.g., `https://login.microsoftonline.com/<non-prod-tenant-id>`).
4. Use **Non-Prod test accounts** for user-facing flows. Do not use your personal production credentials to test OAuth flows in Non-Prod.

### What You Can Do in Non-Prod

- Register redirect URIs for `localhost`, dev, and staging environments.
- Request and test OAuth scopes without admin consent bottlenecks (Non-Prod has a more permissive consent policy for testing).
- Rotate and regenerate client secrets freely.
- Test token acquisition, refresh flows, and error handling end-to-end.
- Validate your app's behavior against Entra-protected APIs using Non-Prod equivalents.

### Non-Prod Limitations to Be Aware Of

- Non-Prod does **not** contain real production users or production data. Test accounts are synthetic.
- Some production-only APIs or Microsoft services may not be available or may behave differently in Non-Prod.
- Non-Prod App Registrations **cannot** be used in production — the tenant IDs, client IDs, and secrets are environment-specific.
- Admin consent grants in Non-Prod do not carry over to Prod.

### Tips for a Smooth Development Experience

- **Treat your Non-Prod App Registration like production from day one.** Use the same redirect URI patterns, scope requests, and secret management practices you intend to use in Prod. This makes promotion much smoother.
- **Document your app's permission requirements early.** Knowing exactly which scopes your app needs — and why — is a prerequisite for the Prod promotion process.
- **Use certificates over client secrets where possible**, even in Non-Prod. It builds the right habit and is required for many Prod workloads.
- **Do not hardcode secrets.** Use environment variables or a secrets manager (e.g., Azure Key Vault) from the start.

---

## Requirements to Promote an App Registration to Prod Entra

Promotion to Prod Entra is a deliberate, reviewed process — not a rubber stamp. The following requirements must be met before a Prod App Registration will be created on your behalf.

### 1. Documented Business Justification
Provide a clear description of what the application does, who uses it, and why it needs to exist in Prod Entra. This becomes part of the permanent record for the App Registration.

### 2. Principle of Least Privilege — Scope Justification
For **every** OAuth scope or API permission your app requests, you must provide a written justification explaining:
- What the permission allows.
- Why your application specifically needs it.
- Why a narrower permission cannot satisfy the requirement.

Requests for broad permissions (e.g., `*.ReadWrite.All`, `Directory.ReadWrite.All`) will be challenged and are unlikely to be approved without a compelling, documented case.

### 3. Designated Application Owner
Every Prod App Registration must have a **named owner** — a person (or team distribution list) who is accountable for the app's lifecycle, secret rotation, and decommissioning. This cannot be a personal account that may leave the organization.

### 4. Secret / Credential Management Plan
You must document how client secrets or certificates will be:
- **Stored** (e.g., Azure Key Vault, a specific secrets manager — not `.env` files or source control).
- **Rotated** (rotation schedule and responsible party).
- **Revoked** in the event of a suspected compromise.

### 5. Redirect URI Review
All redirect URIs must be reviewed and approved. Wildcards are not permitted. Each URI must correspond to a known, controlled endpoint. Localhost and development URIs are not permitted in Prod App Registrations.

### 6. Security Review Sign-Off (for elevated permissions)
Applications requesting admin-consent-level permissions or access to sensitive APIs (e.g., full mailbox access, directory write, privileged identity) require a security review before Prod promotion. Engage the security team early — this is not a last-minute step.

### 7. Validated in Non-Prod
The application must have been fully tested and validated in Non-Prod Entra. Evidence of successful Non-Prod testing (e.g., a working integration, passing test suite, or sign-off from a tech lead) is expected as part of the promotion request.

### 8. Decommission Plan
Know how this app will be retired when it is no longer needed. Prod App Registrations without a decommission plan become tomorrow's orphaned apps. Document the expected lifecycle and who is responsible for cleanup.

---

## Summary: The Developer Workflow at a Glance

```
[New OAuth Integration Needed]
         │
         ▼
[Request Non-Prod App Registration]
         │
         ▼
[Develop & Test Against Non-Prod Entra]
  - Use test accounts
  - Validate all OAuth flows
  - Confirm scope requirements
  - Establish secret management
         │
         ▼
[Prepare Prod Promotion Package]
  - Business justification
  - Scope justification (per permission)
  - Named owner
  - Secret management plan
  - Redirect URI list
  - Non-Prod test evidence
  - Security review (if required)
         │
         ▼
[Submit Prod Promotion Request]
         │
         ▼
[Review & Approval by Identity/Security Team]
         │
         ▼
[Prod App Registration Created — Secrets Delivered Securely]
         │
         ▼
[Deploy to Production]
```

---

## Quick Reference: Common Questions

**Q: Can I just use my personal Azure subscription to test?**
No. Personal Azure subscriptions create a separate, unmanaged tenant that is not connected to our organization's identity infrastructure. OAuth flows that depend on organizational users, groups, or internal APIs will not work correctly, and any app registrations created there are outside our security and compliance boundary.

**Q: How long does the Prod promotion process take?**
Standard applications with well-documented, low-privilege scope requests typically complete review within 3–5 business days. Applications requiring admin consent or a security review may take longer — plan accordingly and do not leave this to the last week before a launch.

**Q: My app needs a permission that requires admin consent. What do I do?**
In Non-Prod, reach out to the Non-Prod Entra admin to grant consent for testing. For Prod, this must be part of your promotion request and will be reviewed by the Identity and Security teams before consent is granted.

**Q: What if my redirect URI changes after the Prod App Registration is created?**
Submit a change request to add or update redirect URIs. Do not attempt to work around this with wildcards or proxy endpoints — both are security risks and will not be approved.

**Q: Can I share a Prod App Registration between multiple applications?**
Generally, no. Each distinct application should have its own App Registration to maintain clear ownership, appropriate scoping, and an independent secret rotation lifecycle. Exceptions require explicit justification and approval.

---

*For questions, intake requests, or to begin the Prod promotion process, contact the Identity Platform team at [your team's contact/channel].*
