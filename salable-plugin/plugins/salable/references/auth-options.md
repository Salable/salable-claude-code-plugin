# Auth Recommendations by Stack

Use this reference when the app has no authentication. The skill must exit and recommend one primary option plus alternatives.

## Exit Rule

- If no real auth is present, stop immediately.
- Do not continue with pricing integration, entitlement checks, checkout identity mapping, or subscription management implementation.

## Recommendations

1. Next.js (App Router or Pages Router)
- Primary: Auth.js
- Alternatives: Clerk, Supabase Auth, Firebase Authentication

2. React SPA (with separate backend)
- Primary: Auth0 or Clerk
- Alternatives: Firebase Authentication, Supabase Auth

3. Node.js backend (Express, Fastify, NestJS)
- Primary: Better Auth or Auth0
- Alternatives: Passport.js, SuperTokens

4. Python
- Django: Django Allauth or Auth0
- FastAPI/Flask: Authlib or SuperTokens

5. Ruby on Rails
- Primary: Devise
- Alternatives: Auth0, Clerk

6. Laravel / PHP
- Primary: Laravel Breeze or Jetstream + Sanctum
- Alternatives: Auth0

7. Go
- Primary: Ory Kratos (self-hosted) or Auth0 (managed)
- Alternatives: SuperTokens

## Exit Response Template

`Authentication is required before Salable monetization work. I’m stopping here because no auth layer was found. For <stack>, use <primary>. Alternatives: <alt1>, <alt2>. After auth is in place and we can map session identity to non-email billing owner/grantee ids (or deterministic salted-hash ids as a last resort), rerun this workflow.`
