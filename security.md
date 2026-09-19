# Security Considerations

This document covers the security practices followed for this project and for the publication of this repository itself.

---

## Practices Followed

- **Never commit secrets.** Credentials, tokens, and keys are kept out of version control entirely — not even in placeholder or commented-out form in real configuration files.
- **Use environment variables / secrets management** for anything sensitive (storage keys, tokens, credentials), rather than embedding them in configuration files.
- **Restrict Loki access.** Loki's API should not be reachable by anyone other than the services and users that need it — it should not be treated as a public-facing service.
- **Restrict log UI access.** The Service Logs Frontend should be limited to authorized/authenticated users, since log content can incidentally include sensitive information.
- **Avoid exposing Loki directly to the public internet.** Access should go through internal networking or an authenticated proxy layer (e.g., Nginx with appropriate access controls), not a direct public endpoint.
- **Sanitize screenshots** before including them anywhere public — see [`screenshots/README.md`](../screenshots/README.md).
- **Mask production domains/IPs** in any example, command, or screenshot.
- **Avoid publishing real logs.** Even seemingly harmless log content can reveal internal naming, request patterns, or infrastructure details.
- **Apply least privilege** for anything that can read from or write to Loki or the object storage backend.
- **Protect Azure Storage credentials.** Storage account keys and SAS tokens are treated as sensitive credentials, scoped as narrowly as possible, and never included in documentation or examples.

---

## Pre-Publication Checklist

Use this checklist before publishing or updating anything in this repository:

```text
[ ] No passwords
[ ] No API tokens
[ ] No Azure keys
[ ] No SAS tokens
[ ] No production IPs
[ ] No internal domains
[ ] No customer data
[ ] No real log samples
[ ] No production .env
[ ] No production compose file
```

---

## Why This Matters for a Public Repository

This repository is meant to demonstrate the design and reasoning behind a real logging system — not to reproduce the system itself. Anything that would let someone reconstruct real infrastructure details (hostnames, storage account names, internal service naming conventions, actual retention figures tied to real volume) is deliberately excluded or replaced with placeholders such as `<PRODUCTION_HOST>`, `<LOKI_HOST>`, `<AZURE_STORAGE_ACCOUNT>`, `<STORAGE_CONTAINER>`, `<INTERNAL_DOMAIN>`, and `<SERVICE_NAME>`.

If in doubt about whether a detail is safe to include, the default is to leave it out or replace it with a placeholder.
