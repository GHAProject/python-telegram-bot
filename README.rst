# Workflow Architecture

## 🏗️ Complete Workflow Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKFLOW TRIGGERS                             │
├─────────────────────────────────────────────────────────────────┤
│  • pull_request (opened, synchronize, reopened)                 │
│  • push (master, main branches)                                 │
│  • release (published)                                          │
│  • schedule (weekly - Sunday midnight)                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     JOB 1: SETUP                                │
├─────────────────────────────────────────────────────────────────┤
│  Action: actions/checkout@v4                                    │
│  Purpose: Initial checkout and validation                       │
│  Output: should_run flag                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┴───────────────────────┐
        ↓                     ↓                       ↓
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  JOB 2: SECURITY │  │  JOB 3: AUDIT    │  │  JOB 4: COVERAGE │
│  & QUALITY       │  │  CHANGES         │  │                  │
├──────────────────┤  ├──────────────────┤  ├──────────────────┤
│ 1. checkout      │  │ 1. checkout      │  │ 1. checkout      │
│ 2. codeql/init   │  │ 2. Create config │  │ 2. Setup Python  │
│ 3. codeql/       │  │ 3. auditor-      │  │ 3. Install deps  │
│    autobuild ✓   │  │    action ✓      │  │ 4. Run tests     │
│ 4. codeql/       │  │                  │  │ 5. coverallsapp/ │
│    analyze       │  │  [PR only]       │  │    action ✓      │
│ 5. lychee-       │  │                  │  │                  │
│    action ✓      │  │                  │  │                  │
│ 6. create-or-    │  │                  │  │                  │
│    update-       │  │                  │  │                  │
│    comment ✓     │  │                  │  │                  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
                              ↓
        ┌─────────────────────┴───────────────────────┐
        ↓                     ↓                       ↓
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  JOB 5: SBOM     │  │  JOB 6: CHANGE-  │  │  JOB 7: RELEASE  │
│  GENERATION      │  │  LOG MGMT        │  │  NOTES           │
├──────────────────┤  ├──────────────────┤  ├──────────────────┤
│ 1. checkout      │  │ 1. checkout      │  │ 1. checkout      │
│ 2. github-sbom-  │  │ 2. Create config │  │ 2. extract-      │
│    generator ✓   │  │ 3. chango ✓      │  │    release-      │
│ 3. Upload        │  │                  │  │    notes ✓       │
│    artifact      │  │  [PR only]       │  │ 3. create-or-    │
│ 4. create-or-    │  │                  │  │    update-       │
│    update-       │  │                  │  │    comment ✓     │
│    comment ✓     │  │                  │  │                  │
└──────────────────┘  └──────────────────┘  │  [Release only]  │
                                            └──────────────────┘
                              ↓
                   ┌──────────────────────┐
                   │  JOB 8: AUTO-UPDATE  │
                   │  PR                  │
                   ├──────────────────────┤
                   │ 1. checkout          │
                   │ 2. Check updates     │
                   │ 3. create-or-update- │
                   │    pull-request ✓    │
                   │                      │
                   │  [Schedule only]     │
                   └──────────────────────┘
                              ↓
                   ┌──────────────────────┐
                   │  JOB 9: WORKFLOW     │
                   │  SUMMARY             │
                   ├──────────────────────┤
                   │ 1. Collect results   │
                   │ 2. create-or-update- │
                   │    comment ✓         │
                   │ 3. Post summary      │
                   │                      │
                   │  [PR only]           │
                   └──────────────────────┘
```

## 📊 Action Distribution by Job

| Job | Actions Used | Count |
|-----|-------------|-------|
| Setup | `checkout` | 1 |
| Security & Quality | `checkout`, `codeql-action/*`, `lychee-action`, `create-or-update-comment` | 4 |
| Audit Changes | `checkout`, `auditor-action` | 2 |
| Test & Coverage | `checkout`, `coverallsapp/github-action` | 2 |
| SBOM Generation | `checkout`, `github-sbom-generator-action`, `create-or-update-comment` | 3 |
| Changelog Management | `checkout`, `chango` | 2 |
| Release Notes | `checkout`, `extract-release-notes`, `create-or-update-comment` | 3 |
| Auto-Update PR | `checkout`, `create-or-update-pull-request-action` | 2 |
| Workflow Summary | `create-or-update-comment` | 1 |

**Total Action Invocations:** 20+ across all jobs

## 🎯 Action Usage Matrix

### Always Used
| Action | Every Job? | Purpose |
|--------|-----------|---------|
| `actions/checkout` | ✅ Yes | Code access |

### Conditional Usage
| Action | When | Condition |
|--------|------|-----------|
| `codeql-action/autobuild` | PRs, Pushes, Weekly | Security scanning |
| `lychee-action` | PRs, Pushes | Link validation |
| `auditor-action` | PRs only | Compliance checks |
| `coverallsapp/github-action` | PRs, Pushes | Coverage reporting |
| `github-sbom-generator-action` | PRs, Pushes | SBOM generation |
| `chango` | PRs only | Changelog fragments |
| `extract-release-notes` | Releases only | Release documentation |
| `create-or-update-pull-request-action` | Weekly | Automated PRs |
| `create-or-update-comment` | PRs, Releases | User feedback |

## 🔄 Data Flow

```
┌─────────────┐
│   Source    │
│    Code     │
└──────┬──────┘
       │
       ↓
┌──────────────────────────────────────┐
│        Parallel Analysis             │
├──────────────────────────────────────┤
│  • Security (CodeQL)                 │
│  • Links (Lychee)                    │
│  • Compliance (Auditor)              │
│  • Coverage (Coveralls)              │
│  • Dependencies (SBOM)               │
│  • Changelog (Chango)                │
└──────┬───────────────────────────────┘
       │
       ↓
┌──────────────────────────────────────┐
│         Artifacts & Reports          │
├──────────────────────────────────────┤
│  • SBOM JSON file                    │
│  • Coverage reports                  │
│  • Security findings                 │
│  • Changelog fragments               │
└──────┬───────────────────────────────┘
       │
       ↓
┌──────────────────────────────────────┐
│      GitHub Integration              │
├──────────────────────────────────────┤
│  • PR Comments (peter-evans)         │
│  • Security Tab (CodeQL)             │
│  • Coverage Dashboard (Coveralls)    │
│  • Releases (extract-release-notes)  │
│  • Auto PRs (gr2m)                   │
└──────────────────────────────────────┘
```

## 🎨 Job Dependencies

```
setup
  ├── security-and-quality ────┐
  ├── audit-changes ───────────┤
  ├── test-and-coverage ───────┼─→ workflow-summary
  ├── generate-sbom ───────────┤
  └── changelog-management ────┘
      │
      ├── release-notes (parallel, release only)
      └── auto-update-pr (parallel, scheduled only)
```

## 🚦 Execution Conditions

### Pull Request Event
- ✅ setup
- ✅ security-and-quality
- ✅ audit-changes
- ✅ test-and-coverage
- ✅ generate-sbom
- ✅ changelog-management
- ✅ workflow-summary
- ❌ release-notes (skipped)
- ❌ auto-update-pr (skipped)

### Push Event
- ✅ setup
- ✅ security-and-quality
- ✅ test-and-coverage
- ✅ generate-sbom
- ❌ audit-changes (skipped)
- ❌ changelog-management (skipped)
- ❌ workflow-summary (skipped)
- ❌ release-notes (skipped)
- ❌ auto-update-pr (skipped)

### Release Event
- ✅ setup
- ✅ release-notes
- ❌ All other jobs (skipped)

### Schedule Event
- ✅ setup
- ✅ security-and-quality
- ✅ test-and-coverage
- ✅ generate-sbom
- ✅ auto-update-pr
- ❌ All other jobs (skipped)

## 📦 Outputs & Artifacts

### File Artifacts
1. **sbom.spdx.json** - Software Bill of Materials
2. **coverage.lcov** - Code coverage report
3. **chango fragments** - Changelog entries

### GitHub Integrations
1. **Security Tab** - CodeQL findings
2. **PR Comments** - Summary reports
3. **Checks Tab** - Status checks
4. **Releases** - Formatted notes

### External Integrations
1. **Coveralls.io** - Coverage dashboard
2. **Dependabot** - Dependency alerts (via SBOM)

## 💡 Optimization Features

### Parallel Execution
Jobs 2-6 run in parallel for speed

### Conditional Execution
- Jobs skip when not applicable
- `continue-on-error` for non-critical checks
- Context-aware triggers

### Resource Efficiency
- Artifacts auto-expire (default 90 days)
- Scheduled runs only on Sunday
- Link checking caches results

### Smart Caching
- Python dependencies cached
- Build artifacts reused
- Git history optimized

## 🎯 Success Criteria

All jobs must pass (except those with `continue-on-error: true`):

| Job | Must Pass? | Impact if Fails |
|-----|-----------|-----------------|
| setup | ✅ Yes | Blocks all jobs |
| security-and-quality | ⚠️ Warn | Comments on PR |
| audit-changes | ⚠️ Warn | Comments on PR |
| test-and-coverage | ✅ Yes | Blocks merge |
| generate-sbom | ✅ Yes | Missing artifact |
| changelog-management | ⚠️ Warn | Manual changelog needed |
| release-notes | ⚠️ Warn | Manual notes needed |
| auto-update-pr | ⚠️ Warn | No auto updates |
| workflow-summary | ⚠️ No | Summary not posted |

---

**Legend:**
- ✓ = Action used in this job
- ✅ = Runs in this scenario
- ❌ = Skipped in this scenario
- ⚠️ = Warning only, doesn't block