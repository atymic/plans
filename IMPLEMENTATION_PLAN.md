# PHP Coverage Report Service — Implementation Plan

> A detailed, issue-by-issue plan for building a Codecov-like PHP coverage service backed by Laravel, designed to be handed to an AI coding agent (Claude Code, Copilot) one issue at a time.

## Architecture Overview

```
┌─ CI ─────────────────────────────────────┐
│                                           │
│  phpunit --coverage-clover=clover.xml     │
│                                           │
│  curl -X POST "$SERVICE_URL/upload" \     │
│    -H "Authorization: Bearer $TOKEN" \    │
│    -F "file=@clover.xml" \                │
│    -F "commit=$SHA" \                     │
│    -F "branch=$BRANCH" \                  │
│    -F "pr=$PR_NUMBER" \                   │
│    -F "slug=$REPO" \                      │
│    -F "flags=unit"                        │
│                                           │
└───────────────────────────────────────────┘
          │ Raw clover.xml (multipart upload)
          │
          ▼
┌─ Laravel ─────────────────────────────────┐
│                                            │
│  1. Receive file + metadata                │
│  2. Parse Clover XML server-side           │
│  3. Normalize paths (strip CI prefix)      │
│  4. Store structured data in DB            │
│  5. Discard the raw XML (don't store it)   │
│  6. Compare against base branch            │
│  7. Post PR comment + commit status        │
│                                            │
└────────────────────────────────────────────┘
```

### Project Structure

```
phpcov/
├── service/          ← Laravel app (the web service)
├── action/           ← GitHub Action wrapper (thin shell script)
└── docs/             ← Documentation site
```

### Data Flow: What We Store vs What We Receive

```
          RECEIVED                              STORED
          ────────                              ──────

     ┌──────────────┐                    ┌──────────────────┐
     │  clover.xml  │    parse &         │  commits         │
     │  (raw file)  │───discard──────►   │  uploads         │
     │              │                    │  file_coverages   │
     └──────────────┘                    │    .path          │
                                         │    .statements    │
     Never stored.                       │    .methods       │
     Never logged.                       │    .method_detail │
     Never touches disk.                 │    .line_coverage │
                                         │    .branches      │
                                         └──────────────────┘

                                         Structured columns.
                                         No raw XML anywhere.
```

### How This Compares to Existing Solutions

| Data              | Coveralls          | Codecov            | SonarQube          | Us                 |
| ----------------- | :----------------: | :----------------: | :----------------: | :----------------: |
| Source code        | ✅ Full source     | ❌                 | ✅ (self-hosted)   | ❌                 |
| Raw coverage XML   | ❌                 | ✅ Stored as blob  | ❌                 | ❌ Parsed & discarded |
| File paths         | ✅                 | ✅ + network list  | ✅                 | ✅ (normalized)    |
| Class/method names | ✅ (in source)     | ✅ (in raw XML)    | ✅                 | ✅ (structured)    |
| Line hit counts    | ✅ per-line array  | ✅ (in raw XML)    | ✅ per-line        | ✅ (structured)    |
| Aggregate metrics  | ✅                 | ✅                 | ✅                 | ✅                 |
| Self-hostable      | ❌                 | Enterprise (paid)  | ✅                 | ✅ Open source     |

---

## Agent Instructions

**Apply to every issue below:**

- Use **Pest** for all tests, not PHPUnit.
- Follow **Laravel 11** conventions.
- Use **PHP 8.3** features: readonly classes, typed properties, named arguments, enums where appropriate.
- Every public method needs a test.
- Reference the code from [`ArtiomTr/jest-coverage-report-action`](https://github.com/ArtiomTr/jest-coverage-report-action) (`formatCoverageSummary.ts`, `generatePRReport.ts`, `checkThreshold.ts`) as inspiration for the comparison, formatting, and threshold logic — but implement in PHP/Laravel idioms, not TypeScript patterns.

---

## Implementation Order

```
Phase 1 (foundation — sequential):   1.1 → 1.2 → 1.3 → 1.4 → 1.5 → 1.6 → 1.7

Phase 2 (core features — sequential): 2.1 → 2.2 → 2.3 → 2.4 → 2.5

Phase 3 (GitHub integration — parallelizable):
  3.1 ─┐
  3.2 ─┼─ (all independent of each other)
  3.3 ─┤
  3.4 ─┘

Phase 4 (polish — parallelizable):
  4.1 ─┐
  4.2 ─┤
  4.3 ─┼─ (all independent)
  4.4 ─┤
  4.5 ─┘
```

---

## Phase 1 — Skeleton + Clover Parsing + Upload Endpoint

**Goal:** `curl` a Clover XML to the service, get back a coverage percentage. No GitHub integration yet. This is the foundation everything else builds on.

---

### Issue 1.1 — Laravel Project Scaffold

Create a new Laravel 11 project in the `service/` directory.

**Requirements:**

- PHP 8.3 minimum
- Laravel 11.x
- SQLite for local dev, MySQL/Postgres support via config
- Sanctum for API token auth
- Queue support (database driver for now, Redis later)
- No Blade/frontend — API only (remove web routes, install Sanctum)

**Packages to install:**

- `laravel/sanctum`
- `pestphp/pest` (for testing, not PHPUnit)

**Config:**

- Set `APP_NAME=PHPCov`
- Create a `config/phpcov.php` with: `max_upload_size` (default `20480` — 20MB in KB), `retention_days` (default `90`)

**Create these empty directories to establish project structure:**

- `app/Services/Parsers/`
- `app/Services/`
- `app/DTOs/`
- `app/Jobs/`
- `app/Http/Controllers/Api/V1/`

**Test:** Write a single Pest test that boots the app and asserts the health check route returns 200.

---

### Issue 1.2 — DTOs for Coverage Data

Create the following DTOs in `app/DTOs/`. These are the internal representation that all parsers produce and all services consume. Use plain readonly PHP 8.3 classes (no third-party DTO library).

**Classes:**

1. **`Metric`** — a simple total/covered pair
   - `int $total`
   - `int $covered`
   - Method `percentage(): float` — returns 0–100, returns `0.0` if total is 0

2. **`MethodDetail`**
   - `string $name`
   - `int $startLine`
   - `int $hits`

3. **`FileCoverageData`**
   - `string $path`
   - `Metric $statements`
   - `Metric $methods`
   - `Metric $branches`
   - `int $complexity`
   - `array $methodDetail` — array of `MethodDetail`
   - `array $lineCoverage` — dense array: `null|int` per line
   - Method `coveragePercentage(): float` — based on statements

4. **`CoverageSummary`**
   - `Metric $statements`
   - `Metric $methods`
   - `Metric $branches`
   - `Metric $lines`
   - `int $totalFiles`
   - Static method `fromFiles(array $files): self` — aggregates all file metrics

5. **`CoverageReport`**
   - `CoverageSummary $summary`
   - `array $files` — array of `FileCoverageData`
   - Method `coveragePercentage(): float`

**Pest tests for:**

- `Metric::percentage()` with 0 total
- `Metric::percentage()` with normal values
- `CoverageSummary::fromFiles()` aggregation
- `CoverageReport::coveragePercentage()`.

---

### Issue 1.3 — Clover XML Parser

Create `app/Services/Parsers/CloverParser.php` that parses Clover XML (as produced by PHPUnit and Pest) into a `CoverageReport` DTO.

**The parser must handle:**

1. **Basic structure:**
   ```xml
   <coverage>
     <project>
       <file name="...">
         <metrics statements="10" coveredstatements="8" .../>
         <line num="5" type="stmt" count="3"/>
         <line num="10" type="method" name="foo" count="2"/>
       </file>
       <metrics .../>
     </project>
   </coverage>
   ```

2. File-level metrics from `<metrics>` element:
   - `statements` / `coveredstatements`
   - `methods` / `coveredmethods`
   - `conditionals` / `coveredconditionals` (these are branches)
   - `complexity`
   - `elements` / `coveredelements`

3. Line-level data from `<line>` elements:
   - `type="stmt"` → statement coverage (lineNum → hitCount)
   - `type="method"` with `name` attribute → method detail
   - `type="cond"` → conditional/branch

4. Dense `line_coverage` array: for a file with max line N, produce an array of length N where index `i` is `null` (non-executable) or `int` (hit count for line `i+1`).

5. Method detail array: `[{name, startLine, hits}]` from `type="method"` lines.

6. Nested packages: some Clover reports put `<file>` inside `<package>` elements. Handle both flat and nested structures.

7. XML safety: use `LIBXML_NONET` flag. Throw `CoverageParseException` (create this exception class in `app/Exceptions/`) on invalid XML.

8. Path handling: preserve the raw path as-is. Path normalization is a separate concern.

**Test fixtures to create:**

- `tests/fixtures/clover-simple.xml` — basic PHPUnit output with 3 files
- `tests/fixtures/clover-nested.xml` — files inside `<package>` elements
- `tests/fixtures/clover-empty.xml` — valid Clover with 0 files
- `tests/fixtures/clover-pest.xml` — Pest-style output

**Pest tests covering:**

- Correct file count extraction
- Correct statement/method/branch metric extraction
- Correct `line_coverage` array (check specific indices for `null` vs `int`)
- Correct method detail extraction (names and hit counts)
- Package-nested files
- Empty project
- Invalid XML throws `CoverageParseException`
- XXE attack attempt is blocked (entity expansion).

---

### Issue 1.4 — Cobertura XML Parser

Create `app/Services/Parsers/CoberturaParser.php` that parses Cobertura XML into the same `CoverageReport` DTO.

PHPUnit supports `--coverage-cobertura` since PHPUnit 9.4. The format:

```xml
<coverage line-rate="0.8" branch-rate="0.6">
  <packages>
    <package>
      <classes>
        <class filename="src/Foo.php" complexity="5">
          <methods>
            <method name="bar" ...>
              <lines><line .../></lines>
            </method>
          </methods>
          <lines>
            <line number="10" hits="3"/>
            <line number="11" hits="0" branch="true"
                  condition-coverage="50% (1/2)"/>
          </lines>
        </class>
      </classes>
    </package>
  </packages>
</coverage>
```

**Handle:**

- Line hits from `<line number="N" hits="H"/>`
- Branch coverage from `condition-coverage="X% (M/N)"` attribute
- Method extraction from `<method>` elements
- Complexity from class-level attribute

**Also create:**

- `app/Services/Parsers/ParserFactory.php`:
  - Static method `make(string $format): ParserInterface`
  - Supports `'clover'` and `'cobertura'`
  - Throws `InvalidArgumentException` for unknown formats

- `app/Services/Parsers/ParserInterface.php`:
  - `parse(string $xml): CoverageReport`

- Format auto-detector:
  - Static method `detect(string $content): string`
  - If contains `<coverage` and `<project` → `'clover'`
  - If contains `<coverage` and `line-rate` → `'cobertura'`
  - Otherwise throw `CoverageParseException`

**Test fixture:** `tests/fixtures/cobertura-simple.xml`

**Pest tests** for parser, factory, and auto-detection.

---

### Issue 1.5 — Database Schema + Models

Create migrations and Eloquent models for the coverage storage layer.

**Migrations (run in this order):**

1. `create_teams_table`:
   - `id`, `name`, `plan` (string, default `'free'`), `timestamps`

2. `create_projects_table`:
   - `id`, `team_id` (FK nullable), `provider` (string 20), `owner`, `name`, `default_branch` (default `'main'`), `webhook_secret` (text nullable), `access_token` (text nullable), `path_strip_prefix` (string nullable), `timestamps`
   - Unique index on `[provider, owner, name]`

3. `create_commits_table`:
   - `id`, `project_id` (FK cascade), `sha` (string 40), `parent_sha` (string 40 nullable), `branch`, `pr_number` (unsigned int nullable), `finalized_at` (timestamp nullable), `timestamps`
   - Unique index on `[project_id, sha]`
   - Index on `[project_id, branch, created_at]`

4. `create_uploads_table`:
   - `id`, `commit_id` (FK cascade), `flag` (string 64, default `'default'`), `ci_name` (string 64 nullable), `lines_total`, `lines_covered`, `statements_total`, `statements_covered`, `methods_total`, `methods_covered`, `branches_total`, `branches_covered`, `complexity` (default 0), `coverage_pct` (decimal 5,2), `timestamps`
   - Index on `[commit_id, flag]`

5. `create_file_coverages_table`:
   - `id`, `upload_id` (FK cascade), `path` (string 500), `lines_total`, `lines_covered`, `statements_total`, `statements_covered`, `methods_total`, `methods_covered`, `branches_total`, `branches_covered`, `complexity` (default 0), `coverage_pct` (decimal 5,2), `method_detail` (json nullable), `line_coverage` (json nullable)
   - Index on `[upload_id, path]`

**Models:**

- `Team` — has many projects
- `Project` — belongs to team, has many commits; `slug()` accessor returning `"{owner}/{name}"`
- `Commit` — belongs to project, has many uploads
- `Upload` — belongs to commit, has many file coverages
- `FileCoverage` — belongs to upload; cast `method_detail` and `line_coverage` as array

**Pest test:** runs all migrations on SQLite and creates a full record chain (team → project → commit → upload → file coverage).

---

### Issue 1.6 — Upload API Endpoint

Create the upload endpoint: `POST /api/v1/upload`

This is the core endpoint. It accepts a multipart form upload with:

| Field    | Type    | Required | Notes                         |
| -------- | ------- | :------: | ----------------------------- |
| `file`   | file    | ✅       | The coverage XML file         |
| `commit` | string  | ✅       | SHA, 40 chars                 |
| `branch` | string  | ✅       | Branch name                   |
| `slug`   | string  | ✅       | owner/repo format             |
| `pr`     | integer | ❌       | PR number                     |
| `flags`  | string  | ❌       | Comma-separated, default `default` |
| `parent` | string  | ❌       | Parent SHA, 40 chars          |
| `service`| string  | ❌       | `github`                      |

**Controller:** `app/Http/Controllers/Api/V1/UploadController.php`

**Flow:**

1. Validate request fields
2. Resolve project from slug + service (create if not exists — auto-provisioning)
3. Authorize via Sanctum token
4. Read file content from upload (in-memory, don't save to disk)
5. Auto-detect format using `ParserFactory::detect()`
6. Parse XML into `CoverageReport` DTO
7. Normalize paths using `PathNormalizer` service (strip common prefix)
8. Create or find `Commit` record (upsert on `project_id` + `sha`)
9. Create `Upload` record with summary metrics
10. Create `FileCoverage` records for each file (bulk insert)
11. Return JSON: `{ "status": "ok", "coverage": 84.5, "files": 42 }`

**Create `app/Services/PathNormalizer.php`:**

- Takes an optional `strip_prefix` from project config
- Finds the longest common directory prefix across all file paths
- Strips it so paths become relative (e.g. `app/Services/Foo.php`)

**Route:** `routes/api.php`, protected by `auth:sanctum` middleware.

**Error responses:**

- 401 if no/invalid token
- 413 if file > 20MB
- 422 if validation fails
- 422 if XML parsing fails (return parser error message)

**Pest tests:**

- Successful upload with Clover fixture, verify DB records created
- Successful upload with Cobertura fixture
- Auto-provisioning creates project on first upload
- Invalid XML returns 422
- Missing required fields return 422
- File too large returns 413
- Path normalization strips `/home/runner/work/app/app/` prefix
- Multiple uploads to same commit with different flags create separate upload records.

---

### Issue 1.7 — Auth: Sanctum Token Setup

Create a simple token management flow.

**1. Artisan command** for creating the first team + project + token:

```bash
php artisan phpcov:create-project {slug} {--service=github}
```

This should:

- Create a `Team` if needed
- Create the `Project` record
- Create a `User` associated with the team
- Generate a Sanctum token
- Print the token to stdout

**2. Authorization:** The upload endpoint should resolve the project from the slug and verify the authenticated user has access to that project's team.

**3. Policy:** `ProjectPolicy@upload` that checks the user belongs to the project's team.

Keep this simple — no registration flow, no OAuth yet. Just tokens for API access. The OAuth/GitHub App flow is a later phase.

**Pest tests:**

- Valid token can upload
- Invalid token gets 401
- Token for wrong team gets 403.

---

## Phase 2 — Comparison Engine + PR Comments

**Goal:** When a PR upload arrives, compare against the base branch and post a comment to GitHub.

---

### Issue 2.1 — Coverage Comparator Service

Create `app/Services/CoverageComparator.php`.

**Methods:**

1. `findBase(Project $project, ?string $parentSha, ?string $branch): ?Upload`
   - First try: exact match on `parent_sha` within project
   - Then try: latest finalized upload on the target branch
   - Then try: latest finalized upload on project's `default_branch`
   - Return `null` if nothing found

2. `compare(Upload $head, ?Upload $base): ComparisonResult`
   - If no base, return a "no baseline" result
   - Key by file path on both sides
   - For each file in head:
     - If not in base → new file
     - If coverage decreased → decreased file
     - If coverage increased → increased file
     - If unchanged → skip
   - Files in base but not in head → removed files
   - Calculate overall coverage diff

**Create `app/DTOs/ComparisonResult.php`:**

- `string $headSha, $baseSha`
- `float $headCoverage, $baseCoverage, $coverageDiff`
- `Metric $statementsDiff, $methodsDiff, $branchesDiff`
- `Collection $newFiles, $removedFiles, $decreasedFiles, $increasedFiles`
- Each is a collection of `FileDiff` DTOs

**Create `app/DTOs/FileDiff.php`:**

- `string $path`
- `?float $headCoverage, $baseCoverage`
- `bool $isNew, $isRemoved`
- Method `diff(): float`

**Pest tests:**

- Comparison with matching files
- New files detection
- Removed files detection
- Decreased coverage detection
- No base available returns appropriate result
- `findBase` priority order (parent SHA > branch > default branch).

---

### Issue 2.2 — Report Formatter (Markdown)

Create `app/Services/ReportFormatter.php`.

Generates markdown for GitHub PR comments, inspired by the `jest-coverage-report-action` format.

**Output structure:**

1. Header with coverage percentage and diff indicator (↑ or ↓ or ±)
2. Summary table: `Status | Category | Percentage | Covered/Total`
   - Statements row
   - Branches row
   - Methods row
   - Lines row
   - Each with 🟢/🟡/🔴 status icon based on threshold
3. If there are decreased files: collapsible `<details>` section listing them with their old → new coverage
4. If there are new files: collapsible `<details>` section listing them with their coverage
5. Footer with "Report generated by 🧪 PHPCov" and link

**Methods:**

- `toMarkdown(ComparisonResult $result, ?float $threshold = null): string`
- `toMarkdownNoBase(Upload $head): string` — first-run, no comparison available

The comment must include a hidden HTML tag `<!-- phpcov-report -->` so we can find and update it on subsequent pushes.

**Pest tests:**

- Markdown contains summary table
- Decreased files appear in details section
- New files appear in details section
- Correct emoji status based on threshold
- Hidden tag is present
- No-base variant works.

---

### Issue 2.3 — GitHub Notifier Service

Create `app/Services/GitHub/GitHubNotifier.php`.

**Methods:**

1. `upsertPrComment(Project $project, int $prNumber, string $markdown): void`
   - GET all comments on the PR issue
   - Find existing comment containing `<!-- phpcov-report -->`
   - If found: PATCH to update it
   - If not found: POST to create it
   - Uses project's `access_token` for authentication

2. `postCommitStatus(Project $project, string $sha, float $coverage, array $thresholdFailures): void`
   - POST to `/repos/{slug}/statuses/{sha}`
   - State: `'success'` if no failures, `'failure'` if any
   - Description: "84.5% coverage" or first failure message
   - Context: "phpcov/coverage"

Use Laravel's `Http` facade. Don't use Octokit or any GitHub SDK.

**Create a contract/interface:**

`app/Contracts/GitProviderNotifier.php`:
- `upsertPrComment(Project, int, string): void`
- `postCommitStatus(Project, string, float, array): void`

This lets us add GitLab/Bitbucket later.

**Pest tests using `Http::fake()`:**

- Creates comment when none exists
- Updates existing comment (finds it by tag)
- Posts success status
- Posts failure status with message.

---

### Issue 2.4 — ProcessCoverageUpload Job (Wiring It Together)

Create `app/Jobs/ProcessCoverageUpload.php`.

This is the async job that runs after an upload is received. Refactor the upload endpoint so it only validates + parses, then dispatches this job.

**The job should:**

1. Receive: project, commitSha, parentSha, branch, prNumber, flag, `CoverageReport` DTO (serialized)
2. Upsert `Commit` record
3. Create `Upload` record with summary metrics
4. Bulk insert `FileCoverage` records (chunk into batches of 100 for large projects)
5. If `prNumber` is set:
   - Find base coverage using `CoverageComparator::findBase()`
   - Compare using `CoverageComparator::compare()`
   - Format markdown using `ReportFormatter`
   - Post PR comment using `GitHubNotifier`
   - Post commit status using `GitHubNotifier`
6. If `prNumber` is null (push to default branch):
   - Post commit status only (no PR comment)

**Update the `UploadController`** to dispatch this job instead of doing DB writes inline. Keep the parse step synchronous (fail fast on bad XML) but move everything after parsing to the job.

Return `202 Accepted` from the controller with the coverage percentage.

**Pest tests:**

- Job creates all DB records correctly
- Job triggers PR comment when prNumber is set
- Job posts commit status
- Job handles missing base gracefully.

---

### Issue 2.5 — Finalize/Notify Endpoint (Multi-Flag Merge)

Create endpoint: `POST /api/v1/repos/{slug}/commits/{sha}/finalize`

This endpoint is called after all parallel CI jobs have uploaded their flags (e.g., unit + integration). It:

1. Marks the `Commit` as finalized (sets `finalized_at`)
2. Merges coverage across all flags for this commit:
   - For each file path across all uploads, take the MAX coverage (union of test suites)
   - Calculate merged summary metrics
3. Runs comparison against base
4. Posts PR comment with merged results
5. Posts commit status

This is optional — if a project only has one flag, the upload endpoint already handles everything. This endpoint is for projects with parallel test suites.

**Create `app/Services/CoverageMerger.php`:**

- `merge(Collection $uploads): Upload` — virtual/unsaved Upload with merged metrics
- For each unique file path, take the upload where that file has the highest coverage percentage
- Recalculate totals from the merged file set.

**Pest tests:**

- Two uploads with different files get unioned
- Two uploads covering the same file take the higher coverage
- Finalize sets `finalized_at` timestamp
- Duplicate finalize calls are idempotent.

---

## Phase 3 — GitHub Action + Badges + Thresholds

**Goal:** A proper GitHub Action wrapper, coverage badges, and configurable thresholds.

---

### Issue 3.1 — GitHub Action (Shell-Based)

Create `action/` directory with a composite GitHub Action.

**`action/action.yml`:**

```yaml
name: 'PHP Coverage Report'
description: 'Upload PHP coverage to PHPCov service'
inputs:
  token:
    description: 'PHPCov API token'
    required: true
  file:
    description: 'Path to coverage file'
    default: 'clover.xml'
  server:
    description: 'PHPCov server URL'
    default: 'https://api.phpcov.dev'
  flags:
    description: 'Comma-separated flags'
    default: 'default'
  finalize:
    description: 'Whether to finalize after upload'
    default: 'true'
outputs:
  coverage:
    description: 'Coverage percentage'
    value: ${{ steps.upload.outputs.coverage }}

runs:
  using: 'composite'
  steps:
    - id: upload
      shell: bash
      run: |
        RESPONSE=$(curl -s -w "\n%{http_code}" -X POST \
          "${{ inputs.server }}/api/v1/upload" \
          -H "Authorization: Bearer ${{ inputs.token }}" \
          -F "file=@${{ inputs.file }}" \
          -F "commit=${{ github.event.pull_request.head.sha || github.sha }}" \
          -F "branch=${{ github.head_ref || github.ref_name }}" \
          -F "slug=${{ github.repository }}" \
          -F "pr=${{ github.event.pull_request.number }}" \
          -F "parent=${{ github.event.pull_request.base.sha }}" \
          -F "flags=${{ inputs.flags }}" \
          -F "service=github")

        HTTP_CODE=$(echo "$RESPONSE" | tail -1)
        BODY=$(echo "$RESPONSE" | sed '$d')

        if [ "$HTTP_CODE" -ge 400 ]; then
          echo "::error::Upload failed with HTTP $HTTP_CODE: $BODY"
          exit 1
        fi

        COVERAGE=$(echo "$BODY" | jq -r '.coverage')
        echo "coverage=$COVERAGE" >> "$GITHUB_OUTPUT"
        echo "✅ Coverage: ${COVERAGE}%"

    - id: finalize
      if: ${{ inputs.finalize == 'true' }}
      shell: bash
      run: |
        SHA="${{ github.event.pull_request.head.sha || github.sha }}"
        curl -s -X POST \
          "${{ inputs.server }}/api/v1/repos/${{ github.repository }}/commits/${SHA}/finalize" \
          -H "Authorization: Bearer ${{ inputs.token }}"
```

**Create `action/README.md`** with usage examples.

---

### Issue 3.2 — Badge Endpoint

Create public (no auth) endpoint: `GET /api/v1/badge/{provider}/{owner}/{repo}`

Returns a coverage badge (shields.io compatible) showing the latest coverage percentage for the default branch.

**Query params:**

- `branch` — optional, override default branch
- `style` — optional: `flat`, `flat-square`, `for-the-badge`

**Implementation:**

- Look up latest finalized upload on the target branch
- 302 redirect to `https://img.shields.io/badge/coverage-{pct}%25-{color}`
- Color: green ≥80%, yellow ≥60%, red <60%
- Cache: set `Cache-Control: max-age=300` (5 minutes).

This endpoint is public — no authentication required.

**Pest tests:**

- Returns redirect with correct percentage
- Returns correct color for different thresholds
- Unknown project returns 404
- Branch override works.

---

### Issue 3.3 — Threshold Configuration + Enforcement

Add threshold support to projects.

**Migration** — add columns to `projects` table:

- `threshold_statements` (decimal 5,2 nullable)
- `threshold_branches` (decimal 5,2 nullable)
- `threshold_methods` (decimal 5,2 nullable)
- `threshold_lines` (decimal 5,2 nullable)
- `threshold_diff` (decimal 5,2 nullable) — max allowed decrease

**Create `app/Services/ThresholdChecker.php`:**

- `check(Upload $upload, Project $project): array` — returns array of failure message strings, empty if all pass
- `checkDiff(ComparisonResult $comparison, Project $project): array` — checks if coverage decreased more than `threshold_diff` allows

**Integrate** into `ProcessCoverageUpload` job.

**API endpoint** to update thresholds: `PATCH /api/v1/projects/{id}`

**Pest tests:**

- Coverage above threshold passes
- Coverage below threshold returns failure message
- Diff threshold catches regressions
- Multiple threshold failures return multiple messages.

---

### Issue 3.4 — Diff-Aware Annotations

Create `app/Services/DiffCoverageAnalyzer.php`.

When a PR upload is processed, fetch the PR diff from GitHub and correlate with `line_coverage` data to find uncovered new/changed lines.

**Flow:**

1. GET `/repos/{slug}/pulls/{pr}/files` from GitHub API
2. Parse the `patch` field to extract added line numbers per file
3. Cross-reference with `line_coverage` from the upload
4. Lines where `line_coverage[lineNum - 1] === 0` AND `lineNum` is in added lines → uncovered new line
5. Check `method_detail`: if a method starts on an added line and has 0 hits, flag it

**Create `app/Services/GitHub/GitHubAnnotator.php`:**

- Post annotations using the Check Runs API

**Pest tests using `Http::fake()`:**

- Correctly identifies uncovered added lines
- Does not flag uncovered lines that weren't changed
- Correctly parses unified diff format
- Posts annotations to GitHub API.

---

## Phase 4 — Polish + Open Source Ready

---

### Issue 4.1 — Coverage History Endpoint

Create endpoint: `GET /api/v1/repos/{slug}/history`

Returns coverage over time for a branch (for sparkline/trend graphs).

**Query params:** `branch`, `days` (default 30, max 365), `per_page` (default 100)

**Response:**

```json
{
  "data": [
    { "sha": "abc123...", "coverage": 84.5, "date": "2026-02-20T..." },
    { "sha": "def456...", "coverage": 83.2, "date": "2026-02-19T..." }
  ]
}
```

Only return finalized commits on the specified branch. Order by `created_at` descending.

---

### Issue 4.2 — Retention + Cleanup

Create `app/Console/Commands/CleanupOldCoverage.php`.

Scheduled command that deletes data older than the configured retention period.

- Delete old `file_coverages`, `uploads`, empty `commits`
- Keep the latest upload per branch regardless of age (so badges always work)
- Schedule: daily at 3am UTC
- Use `config('phpcov.retention_days')` — default 90

---

### Issue 4.3 — GitHub Webhook Receiver

Create endpoint: `POST /api/v1/webhooks/github` (public, no Sanctum auth)

Handle `pull_request.opened` / `pull_request.synchronize` events. Verify webhook signature. Return 200 immediately.

---

### Issue 4.4 — Documentation + README

Root README, service README, action README, CONTRIBUTING.md, LICENSE (MIT).

---

### Issue 4.5 — Docker + Self-Hosting

`docker-compose.yml`, `Dockerfile` (PHP 8.3 FPM Alpine), `.env.example`, `Makefile`.