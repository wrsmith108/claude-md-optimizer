---
description: "JWT RS256 auth (15-min access tokens), bcrypt cost 12, rate limits (5 req/15min auth, 100 req/min API), zod input validation, no secrets in source control"
loading_strategy: "lazy"
---

# Security Rules

## Secrets

- **NEVER** commit API keys, tokens, or credentials to source control
- Use `.env.local` (gitignored) for local development
- Production secrets injected via environment at deploy time

## Authentication

- All auth uses JWT with RS256 signing
- Token expiry: **15 minutes** (access), **7 days** (refresh)
- Store refresh tokens in httpOnly cookies, never in localStorage

## Password Hashing

- Use `bcrypt` with cost factor **12**
- Never store plaintext or MD5/SHA hashed passwords

## Rate Limiting

- Auth endpoints: 5 requests / 15 minutes per IP
- API endpoints: 100 requests / minute per user
- Implemented via `express-rate-limit` + Redis store

## Input Validation

- Validate and sanitize all user input at the API boundary using `zod`
- Parameterized queries only — no string interpolation in SQL
- Sanitize HTML output with `DOMPurify` on the client
