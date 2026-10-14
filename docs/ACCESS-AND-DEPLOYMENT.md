# JacSYS — JacMIN Access Control and Deployment Design

**Status:** Draft for architectural review — not deployment instructions to run unattended  
**Version:** 0.1 | **Prepared:** 13 October 2026  
**Target hosts:** UQ Cloud webproject-style zone and a suitable Raspberry Pi/Linux server

## 1. Purpose

Specify secure identity, permission and runtime requirements for **JacMIN**, the sole administrative producer of the public JacSON/indexSON static API. JacDASH, JacBLACK and future JacXEL remain separately deployed **read-only** consumers of public API artifacts. This is an architecture/handover document: Codex must produce step-by-step, user-executed deployment guides before any deployment occurs.

## 2. Design principles

1. **Authenticate at the edge** where appropriate, **authorise in the application** on every privileged action.
2. **System administrators are configured outside self-service user management.** Lower roles are delegable only by those preconfigured admins; delegated users cannot escalate or delegate permissions.
3. Keep authorisation decisions server-side; do not trust role selectors, hidden buttons or client-side flags.
4. Minimise secrets and privileges. Publishing credentials must not be exposed to the browser or static API.
5. Prefer one codebase with clear **host-specific authentication adapters**, rather than making the scraper depend on UQ Cloud.
6. Never require JacMIN to stay online for published consumer data to work.
7. Do not give Codex permission to deploy, create zones, change DNS, issue credentials or change GitHub repo settings during planning.

## 3. Role model — proposed baseline

| Capability | Viewer | Editor / manager | Publisher | System admin |
|---|:---:|:---:|:---:|:---:|
| Search, inspect source and history | ✓ | ✓ | ✓ | ✓ |
| Edit drafts / set manual mode | — | ✓ | Configurable | ✓ |
| Trigger targeted scrape | — | ✓* | ✓ | ✓ |
| Trigger all-UQ scrape/reconciliation | — | — | Configurable | ✓ |
| Approve and publish to public API | — | — | ✓ | ✓ |
| Configure schedules / credentials | — | — | — | ✓ |
| Add/revoke non-admin users | — | — | — | ✓ |
| Grant system-admin status | — | — | — | **Not through UI** |

`*` Scoped/rate-limited to prevent abuse. The role model must be reviewed with the project owner and institutional policy. A public JacDASH user is not necessarily a JacMIN “viewer” account.

### Administrator anchoring

- Maintain a trusted list of **core system admins** in deployment-managed configuration (not editable through JacMIN web UI).
- A system admin may invite/add a named UQ user to a less-privileged role, and may revoke that grant.
- Added users cannot add other users or grant themselves administrative privileges. User management routes must enforce the policy server-side.
- Record role changes privately and retain an audit trail.
- Avoid generating a writable nginx `auth.conf` file directly from browser-submitted user strings. If access rules at the reverse proxy must change, validate and atomically stage changes using a privileged, narrowly scoped helper, then test config and reload safely. Prefer to keep dynamic lower-role authorisation inside JacMIN instead.

## 4. UQ Cloud authentication: source-backed facts

The provided **UQcloud Zones Guide** describes the `webproject` image as an nginx-fronted application environment with built-in UQ SSO via nginx + `webacld`. The guide identifies:

- ACL mapping through `/etc/nginx/conf.d/auth.conf`;
- authenticated UQ user info in proxy-controlled `X-UQ-User`, `X-UQ-User-Email` and `X-KVD-Payload` headers;
- Python web framework deployment through `uwsgi`/WSGI behind nginx;
- UQ Cloud's reverse/load-balancer HTTPS termination;
- a `webprojctl` tool to enable appropriate services.

The guide notes the SSO headers are protected against spoofing **in the documented trusted nginx/uwsgi deployment pattern**. Treat them as untrusted if JacMIN is exposed directly, deployed behind an arbitrary new proxy, or configured to accept user-injected headers.

### Recommended UQ Cloud adapter

1. Require SSO at nginx for administrative routes.
2. Ensure JacMIN is accessible only through the trusted web server/proxy and is bound to a loopback socket where appropriate.
3. Extract the authenticated UQ username from the verified server-side request context; map to JacMIN roles in its own policy store.
4. Deny access by default and record denied administrative operations.
5. Confirm HTTPS and trusted proxy header configuration as deployed; do not blindly reuse development settings in production.

An existing basic SSO proof-of-concept was reported as successfully online in earlier JacSYS discussions. Codex should inspect its deployment notes/configuration if provided rather than reconstructing the working setup from assumptions.

## 5. Raspberry Pi / general Linux authentication

**Do not assume UQ SSO magically works outside UQ Cloud.** A Raspberry Pi should use one of these reviewed choices:

- approved institutional OIDC/SSO integration where registration, host access and policies explicitly allow it;
- a secure reverse proxy that provides independently verified identities;
- a local administrative login solution with suitable password hashing, MFA where available, lockout/rate limits, TLS and recovery controls.

This is a **planning decision**, not an instruction to invent or implement an unofficial UQ SSO credential collection flow. Never ask users to enter UQ passwords into an app-controlled login page masquerading as institutional SSO.

For a Pi deployed on a private network, require explicit control of who can reach it (firewall/VPN/Tailscale or equivalent as approved), not merely a strong password. Public exposure needs additional review.

## 6. Deployment interface and modularity

JacMIN should expose an importable, UI-independent engine and CLI, such as conceptual commands:

```bash
jacmin scrape --course MGTS2301 --semester 7660
jacmin catalogue refresh --dry-run
jacmin index rebuild --dry-run
jacmin api validate --staging
jacmin api publish --release <approved-release-id>
```

**These are target design examples, not commands that currently exist.** The web UI, scheduler and CLI invoke the same validated job engine. Scrapes and publishing must run as background jobs independent of browser sessions; lock conflicting jobs.

Run application services as non-root identities, with read/write permissions limited to operational data. Use separate public static assets and private operational state. Control startup/restart via the host's supported service manager; produce a proper health check and diagnostics endpoint that does not leak credentials.

## 7. UQ Cloud environment checklist

Codex's deployment manual must include **user-executed** steps for:

- confirming zone owner, image, package/quota, DNS and network selection;
- identifying `webproject` version and actual Python runtime/uwsgi compatibility;
- locating existing nginx SSO and ACL configuration, taking backups before changes;
- installing JacMIN in an isolated Python environment without modifying system Python;
- configuring app service, base URL, filesystem access, logs and persistent storage;
- binding service to local/trusted interface only; validating trusted SSO headers;
- testing access as a configured admin, delegated editor and denied user;
- testing outgoing requests to course profiles, catalogue pages and GitHub independently;
- configuring GitHub publishing credentials using least privilege;
- setting up user-reviewed schedule and monitored backups;
- showing how to stop, rollback and uninstall without destroying unrelated zone services.

**Important UQ Cloud guide caveats:** the guide warns about special permissions under `/var/www` (avoid blanket `chmod -R 777`), zone snapshots and account-owner lifecycle. Long-lived institutional production use should consider a suitable **group/project account** rather than a personally owned zone. Verify all guide instructions against the **actual** image/version before deployment.

## 8. Raspberry Pi / generic Linux environment checklist

Codex should provide separate installation instructions for a supported ARM64 OS and adequate storage, CPU, memory and reliability. Cover:

- user/system packages, virtual environment, service manager and non-root process;
- persistent operational database and a separate writable staging directory;
- permission-safe config/secret storage and rotation;
- protected admin web access, authentication adapter and TLS/network policy;
- headless operation, monitoring, reboot recovery, backups and tested restore;
- dedicated publisher lock and reliable connectivity to GitHub;
- outage behaviour (continue serving last published data even if the Pi fails);
- planned migration path between Pi and UQ Cloud (export/import of operational state with encrypted backups).

No requirement to run large in-memory datasets or a heavyweight database for a single-site deployment without a justified measurement.

## 9. Remote source/network matrix — required test

| Network host | `course-profiles.uq.edu.au` | `programs-courses.uq.edu.au` | GitHub API/push |
|---|---|---|---|
| GitHub Actions | README/log states profile scraper works | Offering discovery script says 405 on cloud ranges | Expected, verify permissions |
| UQ Cloud zone | **Test** | **Test** | **Test** |
| Raspberry Pi/home network | **Test** | **Test** | **Test** |
| Permitted local workstation | **Test** | Existing script recommends local run | **Test** |

If catalogue discovery is blocked, preserve last known-good indexSON catalogue data, signal the problem, and use approved acquisition methods. Do not implement undocumented access-control circumvention.

## 10. Secret and supply-chain controls

- Use least-privilege GitHub credentials; restrict to publishing repo/branch/workflow where possible.
- Never commit real tokens, UQ passwords, service account JSON, `.env`, login cookies or operational database files.
- Replace legacy parsing patterns such as `eval(credentials_json)` with proper JSON parsing and restricted configuration loading; do not carry unsafe convenience code into JacMIN.
- Dependency pinning/updates, automated security checks, linting, logging redaction and branch protection must be part of the plan.
- Do not expose source HTML, admin notes or internal staff identifiers in publicly indexed artifacts unless verified to be public and necessary.
- Enforce CSRF protection, session expiry, secure cookies, input/output sanitisation and protection against path traversal, SSRF and GitHub token leakage.
- A manual-edit preview must not blindly render arbitrary user-supplied HTML/JavaScript; JacBLACK similarly sanitises rich source content.

## 11. Backups and recovery

API Git history is useful but **not sufficient** as a JacMIN operational backup. Plan separate backup/restore of:

- audited manual overrides and drafts;
- latest validated automatic source snapshots;
- job/release history;
- role grants and administrator list/configuration;
- secrets (stored via approved secure mechanism, not necessarily in the same backup).

Define recovery point/restore-time objectives for the project, then test restoring onto a fresh Pi and/or zone. Keep at least one known-good published API release available when JacMIN is down.

## 12. Security acceptance scenarios

1. Unauthenticated user cannot access JacMIN routes or APIs; public JacDASH/API remain accessible as intended.
2. Delegated editor cannot add users, publish without permission, or elevate their role via raw HTTP request.
3. User-forged `X-UQ-User` or `X-KVD-Payload` sent directly to an untrusted interface cannot gain privileges.
4. Changing app/UI role flags has no effect on server-side permission checks.
5. Secrets do not appear in logs, error responses or Git commits.
6. Pi reboot resumes scheduler safely without duplicating a running publication job.
7. UQ Cloud restart/upgrade retains overrides and audit state.
8. When JacMIN is unreachable, pinned API/Blackboard content remains retrievable.
9. Failure to contact programme catalogue does not erase programme indexes.
10. Publisher account can write only required resources, not administer every repository.

## 13. Decisions requiring institutional/project owner confirmation

- **AUTH-01:** UQ Cloud zone image and confirmed available UQ SSO/proxy trust pattern.
- **AUTH-02:** Required roles and which actions are authorised to editors versus publishers.
- **AUTH-03:** Approved authentication mechanism for Raspberry Pi and whether off-campus access is required.
- **AUTH-04:** Hosting account ownership, ongoing stewardship and backups after staff changes.
- **AUTH-05:** Public API content/privacy/classification review, particularly optional UQBS internal overlays.
- **AUTH-06:** GitHub publication identity, permitted secrets system and target branch protection.
- **AUTH-07:** Approval for catalogue access from chosen host and scheduled request volume.

## 14. References

- Uploaded **UQcloud zones guide.pdf**, particularly sections on zones, `webproject`, nginx/SSO/`X-KVD-Payload`, Python/uwsgi and group accounts.
- Prior JacSYS UQ Cloud SSO proof-of-concept and administrator-role decisions (ask owner for its deployment directory before implementation).
- [Sean's offering discovery source](https://github.com/uqgblaze/uqbs-course-profiles/blob/main/scraper/discover_offerings.py) (405/network caveat).
- Uploaded legacy `update_status.py` and `generate_course_list.py` (uses `eval` for service-account JSON; pattern must be retired).
