# Proposal Numbering Migration Guide

This guide contains the migration instructions for existing open proposal PRs to adopt the new PR-number-based numbering system.

## Background

We've adopted a new proposal numbering system where proposal numbers match their PR numbers. This eliminates collisions and simplifies the workflow.

## Migration Instructions by PR

### PR #70 - Routing API
**Current file:** `proposals/004-routing-api.md`  
**New file:** `proposals/070-routing-api.md`

```bash
git mv proposals/004-routing-api.md proposals/070-routing-api.md
git commit -m "Rename proposal to use PR number 70"
git push
```

### PR #82 - Kroxylicious 1.0
**Current file:** `proposals/007-kroxylicious-1.0.md`  
**New file:** `proposals/082-kroxylicious-1.0.md`

```bash
git mv proposals/007-kroxylicious-1.0.md proposals/082-kroxylicious-1.0.md
git commit -m "Rename proposal to use PR number 82"
git push
```

### PR #83 - Hot Reload Feature
**Current file:** `proposals/012-hot-reload-feature.md`  
**New file:** `proposals/083-hot-reload-feature.md`

```bash
git mv proposals/012-hot-reload-feature.md proposals/083-hot-reload-feature.md
git commit -m "Rename proposal to use PR number 83"
git push
```

### PR #85 - Audit Logging
**Current file:** `proposals/nnn-audit-logging.md`  
**New file:** `proposals/085-audit-logging.md`

```bash
git mv proposals/nnn-audit-logging.md proposals/085-audit-logging.md
git commit -m "Rename proposal to use PR number 85"
git push
```

### PR #88 - Frontend Handler Refactoring
**Current file:** `proposals/014-frontend-handler-refactoring-and-centralised-session-context.md`  
**New file:** `proposals/088-frontend-handler-refactoring-and-centralised-session-context.md`

```bash
git mv proposals/014-frontend-handler-refactoring-and-centralised-session-context.md proposals/088-frontend-handler-refactoring-and-centralised-session-context.md
git commit -m "Rename proposal to use PR number 88"
git push
```

### PR #93 - Correlated Request/Response Data API
**Current file:** `proposals/xxx-correlated-request-response-data-api.md`  
**New file:** `proposals/093-correlated-request-response-data-api.md`

```bash
git mv proposals/xxx-correlated-request-response-data-api.md proposals/093-correlated-request-response-data-api.md
git commit -m "Rename proposal to use PR number 93"
git push
```

### PR #94 - TLS Configuration Deprecation
**Current file:** `proposals/nnn-tls-configuration.md`  
**New file:** `proposals/094-tls-configuration.md`

```bash
git mv proposals/nnn-tls-configuration.md proposals/094-tls-configuration.md
git commit -m "Rename proposal to use PR number 94"
git push
```

## Comment Template for PR Authors

Use this template to comment on each PR:

---

Hi! We've updated the proposal numbering system to use PR numbers instead of sequential allocation. This eliminates number collisions and simplifies the workflow.

**Action required:** Please rename your proposal file to use this PR's number.

**Current file:** `proposals/XXX-name.md`  
**New file:** `proposals/0YY-name.md`

```bash
git mv proposals/XXX-name.md proposals/0YY-name.md
git commit -m "Rename proposal to use PR number YY"
git push
```

See [proposals/README.md](https://github.com/kroxylicious/design/blob/main/proposals/README.md) for the updated workflow. Going forward, a GitHub Action will automatically remind you if the filename doesn't match the PR number.

---

## After Migration

Once all open PRs have been renamed:
- The GitHub workflow will automatically check new proposals
- No manual index maintenance is needed - the directory listing is the index
- Proposal numbers will match PR numbers going forward
