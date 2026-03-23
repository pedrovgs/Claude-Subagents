# Analyse Tests Quality

Analyze test coverage, test design quality, and product risk across packages. Generate an HTML report identifying blind spots where missing tests could cause user-visible bugs.

## Input

$ARGUMENTS - Space-separated package paths to analyze

## Execution Model

This command has three phases:
1. **Per-package analysis** — delegated to individual subagents (parallelized)
2. **Aggregation + report** — done by the main agent after all subagents complete
3. **Output collection** — copy all artifacts to `~/Desktop/test-quality-report/`

### Step 1: Spawn one subagent per package

For each package path in $ARGUMENTS, spawn an **Agent subagent** with the prompt below. Run all subagents **in parallel**.

Each subagent receives ONE package path and returns a **structured JSON block** with all analysis results.

#### Subagent Prompt Template

```
Analyze test quality for the package at: {PACKAGE_PATH}

Return your findings as a single JSON code block (```json ... ```) with this structure:

{
  "package_path": "{PACKAGE_PATH}",
  "language": "swift|typescript|python|go|kotlin",
  "exists": true,
  "metrics": {
    "production_files": 0,
    "test_files": 0,
    "test_cases": 0,
    "production_loc": 0,
    "test_loc": 0
  },
  "coverage": {
    "available": false,
    "generated": false,
    "generation_failed": false,
    "failure_reason": null,
    "line_pct": null,
    "function_pct": null,
    "zero_coverage_files": [],
    "low_coverage_files": [],
    "original_report_paths": [],
    "per_file": []
  },
  "critical_uncovered_sections": [],
  "blind_spots": [
    {
      "file": "path/to/file.swift",
      "description": "Public API without tests",
      "category": "public_api|core_logic|complex_file|error_handling|persistence|state_management",
      "severity": "CRITICAL|HIGH|MEDIUM|LOW",
      "user_impact": "If X breaks → user sees Y",
      "impact_example": "e.g. user taps Save but document silently fails to persist, losing all recent edits"
    }
  ],
  "test_quality": {
    "strengths": ["..."],
    "weaknesses": ["..."],
    "suggestions": ["..."]
  },
  "coverage_by_test_type": {
    "unit": { "count": 0, "line_pct": null },
    "integration": { "count": 0, "line_pct": null },
    "other": { "count": 0, "line_pct": null }
  },
  "complexity_risks": [
    {
      "file": "path/to/file.swift",
      "loc": 0,
      "conditionals": 0,
      "has_tests": false
    }
  ]
}

Follow these instructions for the analysis:

### Discovery

1. **Detect language/framework**:
   - Swift: look for `Package.swift`, `*.xcodeproj`, `*.swift` files, XCTest imports
   - TypeScript/JavaScript: look for `package.json`, jest/vitest/mocha config
   - Python: look for `setup.py`, `pyproject.toml`, pytest config
   - Go: look for `go.mod`
   - Kotlin: look for `build.gradle.kts`, `build.gradle`

2. **Identify production vs test files**:
   - Test patterns: `*Test.swift`, `*Tests.swift`, `*Spec.swift`, `*.test.ts`, `*.spec.ts`, `test_*.py`, `*_test.go`, files under `Tests/`, `__tests__/`, `tests/`, `test/`
   - Exclude: build folders, dependencies (`node_modules`, `.build`, `Pods`, `DerivedData`, `vendor`, `dist`)

3. **Count LOC** for production and test files separately (non-blank, non-comment lines)

### Metrics Collection

| Metric | How |
|--------|-----|
| Production files count | Count non-test source files |
| Test files count | Count test files |
| Test cases count | Count `func test*`, `it(`, `test(`, `describe(`, `def test_`, `func Test*` |
| Production LOC | Non-blank lines in production files |
| Test LOC | Non-blank lines in test files |

### Coverage Collection (MANDATORY)

Coverage data is **required**. The subagent must attempt to generate it if not already present.

#### Step A: Look for existing coverage reports
- Swift: `.xcresult`, `codecov.json`, `lcov.info` (check inside `.build/` for SPM)
- JS/TS: `coverage/lcov.info`, `coverage/coverage-summary.json`
- Python: `.coverage`, `htmlcov/`, `coverage.xml`
- Go: `coverage.out`

**Freshness check:** If an existing coverage report is found, check its modification time (`stat -f %m` on macOS or `stat -c %Y` on Linux). If the report was modified **less than 24 hours ago**, reuse it — do NOT regenerate. Only regenerate if the report is older than 24 hours or if the user explicitly requested fresh coverage (e.g. `--force-coverage` flag or explicit instruction).

#### Step B: If no existing reports found OR existing reports are older than 24 hours, attempt to generate coverage

Run the appropriate coverage command based on detected language:

| Language | Command |
|----------|---------|
| Swift (SPM) | Try `swift test --enable-code-coverage` first. If it fails (e.g. UIKit dependency), fall back to **iOS Simulator** approach (see below). |
| Swift (Xcode) | `xcodebuild test -scheme {scheme} -enableCodeCoverage YES -resultBundlePath coverage.xcresult` |
| JS/TS (Jest) | `npx jest --coverage --coverageReporters=json-summary --coverageReporters=lcov` |
| JS/TS (Vitest) | `npx vitest run --coverage` |
| Python | `python -m pytest --cov=. --cov-report=json --cov-report=lcov` |
| Go | `go test -coverprofile=coverage.out -covermode=count ./...` |

**Swift iOS Simulator fallback (for packages with UIKit/platform dependencies):**

When `swift test` fails due to missing UIKit or platform-specific frameworks, use `xcodebuild` targeting an iOS Simulator:

1. **Find an available simulator**: `xcrun simctl list devices available | grep "iPhone" | head -1` to get a device ID
2. **Run tests with coverage**:
   ```
   xcodebuild test \
     -scheme {PackageName} \
     -destination 'platform=iOS Simulator,id={SIMULATOR_ID}' \
     -enableCodeCoverage YES \
     -derivedDataPath .build/derivedData \
     -resultBundlePath .build/coverage.xcresult \
     2>&1
   ```
   Where `{PackageName}` is the SPM package name (from Package.swift `name:` field).
3. **Extract coverage from xcresult**: Use `xcrun xccov view --report --json .build/coverage.xcresult` to get JSON coverage data. Parse it to extract per-file line/function coverage.
4. **Record the xcresult path** in `coverage.original_report_paths`.

**Important constraints for coverage generation:**
- Timeout: 10 minutes max per package (iOS Simulator builds take longer)
- Do NOT install dependencies/tools that aren't already available
- Do NOT modify source code
- If the package has no test files, skip coverage generation (set `generation_failed: true`, `failure_reason: "No test files found"`)

#### Step C: If generation fails

Set in the JSON:
```json
"coverage": {
  "available": false,
  "generated": false,
  "generation_failed": true,
  "failure_reason": "Exact error message or reason why coverage could not be collected",
  ...
}
```

Common failure reasons to capture:
- "No test files found"
- "swift test failed: {error}"
- "Coverage tooling not installed (missing jest/pytest/etc)"
- "Build failed: {error}"
- "Timeout exceeded (5 min)"
- "No compatible test runner detected"

#### Step D: If coverage is available (pre-existing or freshly generated)

1. Parse and extract line/function coverage %, files with 0% and <30% coverage.
2. **Record the absolute paths** of ALL coverage report files/directories into `coverage.original_report_paths`.
3. Set `coverage.available = true` and `coverage.generated = true/false` (true if you generated it, false if pre-existing).
4. **Extract per-file coverage** into `coverage.per_file` — an array of objects, one per production source file:
   ```json
   {
     "file": "relative/path/to/File.swift",
     "line_pct": 45.2,
     "function_pct": 55.0,
     "loc": 250,
     "uncovered_lines": 137
   }
   ```
   This data powers the interactive coverage treemap in the HTML report. Include ALL production files, even those at 100%.

### Critical Uncovered Sections Per Package (Top 10)

After coverage is collected, identify the **top 10 most critical code sections with zero or near-zero coverage**. For each, read the actual source code to extract the specific function/method/class that is uncovered.

Populate `critical_uncovered_sections` with up to 10 entries:
```json
{
  "rank": 1,
  "file": "Sources/MyModule/CoreEngine.swift",
  "absolute_path": "/Users/pedro/Development/Projects/*",
  "line_start": 45,
  "line_end": 92,
  "symbol": "func processTransaction(_ tx: Transaction) -> Result<Receipt, Error>",
  "description": "Core transaction processing with no coverage",
  "line_pct": 0.0,
  "severity": "CRITICAL",
  "user_impact": "If transaction processing breaks → user loses payment data",
  "impact_example": "User completes a purchase but the receipt is never generated, leaving them with no proof of payment and a stuck order state",
  "loc": 47
}
```

**Ranking criteria** (in priority order):
1. Severity (CRITICAL > HIGH > MEDIUM > LOW)
2. Lower coverage % is worse
3. Higher LOC = more risk
4. More conditionals = more risk

Each entry must include the actual function/method signature from the source code (the `symbol` field) so the report can link directly to it.

### Blind Spot Detection

Identify risky areas:
1. **Public APIs without tests**: public classes/functions/methods with no corresponding test
2. **Core domain logic uncovered**: files in core/domain/model paths without tests
3. **Complex files without tests**: files >500 LOC or many conditionals (`if`, `guard`, `switch`, `when`, `?:`) without test counterparts
4. **Error handling paths**: `catch`, `throw`, `Error`, `Result.failure` paths without coverage
5. **Data persistence logic**: files touching databases, file I/O, UserDefaults, CoreData, Realm without tests
6. **State management**: files managing app state (reducers, stores, managers) without tests

For each blind spot, write:
- A 1-line user impact: "If X breaks → user sees Y"
- A concrete impact example: a specific scenario describing the bug or UX degradation the user would experience (e.g. "user taps Export PDF but gets a blank file because the render pipeline silently skips pages with mixed content")

Classify severity:
- CRITICAL: financial calc, auth/security, persistence, critical domain logic
- HIGH: user workflows, important APIs, state management
- MEDIUM: supporting logic, validation helpers
- LOW: formatting, logging, internal utilities

### Test Design Quality

**Weak patterns** (flag):
- Tests with 0 or 1 assertion
- Test methods >50 lines
- Only happy-path tests
- Excessive mocking (>5 mocks per test)
- Implementation-coupled tests
- Duplicate tests

**Good patterns** (acknowledge):
- Edge case coverage
- Negative/error tests
- Behavior-based assertions
- Integration tests
- Clear naming (given/when/then)
- Test helpers/fixtures

Produce: strengths list, weaknesses list, top 3 suggestions.

### Complexity Risk

Flag files that are complex AND untested:
- >500 LOC without tests
- >15 conditionals without tests
- >4 nesting levels without tests

### Test Type Classification

Classify each test file as **unit**, **integration**, or **other**:
- **Unit**: tests a single class/function in isolation, minimal or no dependencies on other modules, may use simple stubs/mocks
- **Integration**: tests interaction between multiple components, uses real dependencies (databases, network, file system, CoreData), or tests end-to-end flows
- **Other**: performance tests, snapshot tests, UI tests, or tests that don't fit the above categories

Heuristics for classification:
- Files importing only the module under test → unit
- Files importing multiple internal modules or external frameworks (UIKit, CoreData, XCTest+async expectations) → integration
- Files with `measure {}` blocks → other (performance)
- Files under `UITests/` or with `UITest` in name → other (UI)

Populate `coverage_by_test_type` with the count of test cases per type. If per-type coverage can be determined (e.g., by running subsets), include `line_pct`; otherwise set `line_pct: null`.

### Important
- Be thorough but practical — sample large packages (>200 files) focusing on public APIs, core logic, complex files
- If package path doesn't exist, set `exists: false` and return minimal JSON
- Coverage generation is allowed and encouraged — run test commands in read-only mode (don't modify source)
```

### Step 2: Collect results and check coverage

Wait for all subagents to complete. Parse the JSON block from each subagent's response.

**Coverage gate:** For each package where `coverage.generation_failed == true`:
1. **Stop** before generating the HTML report
2. **Notify the user** with a clear message listing each package that failed and the reason:
   ```
   Coverage could not be collected for the following packages:
   - {package_path}: {failure_reason}
   - {package_path}: {failure_reason}
   ```
3. **Ask the user** whether to:
   - a) Continue without coverage for those packages (report will show "Coverage unavailable" with the reason)
   - b) Abort the report generation entirely
4. Only proceed to Step 3 after the user responds

If all packages have `coverage.available == true`, proceed directly to Step 2b.

### Step 2b: Merge cross-package coverage

Packages that depend on each other may produce coverage for files outside their own source tree. For example, if package B depends on package A, running B's tests may also exercise code in A. This step merges those overlapping results so each file's coverage reflects the **best** (maximum) achieved across all packages.

1. **Build a file coverage index**: Iterate over every package's `coverage.per_file` entries and group them by **absolute file path** (resolve relative paths using the package's `package_path` as base).
2. **Detect overlaps**: A file appears in multiple packages when its resolved path matches across two or more `per_file` arrays.
3. **Merge strategy — take the maximum per file**:
   - For each file that appears in N packages, produce a single merged entry:
     - `line_pct` = max(`line_pct`) across all appearances
     - `function_pct` = max(`function_pct`) across all appearances
     - `uncovered_lines` = min(`uncovered_lines`) across all appearances
     - `loc` stays the same (it's a file property, not coverage-dependent)
   - Keep track of which packages contributed coverage for each file (store as `covered_by: [pkg1, pkg2, ...]`).
4. **Update each package's results**:
   - For each package, replace `coverage.per_file` entries with the merged values for any overlapping files.
   - Recalculate the package-level `coverage.line_pct` as the weighted average of all its `per_file` entries (weighted by `loc`).
   - Recalculate `coverage.function_pct` similarly.
   - Recalculate `coverage.zero_coverage_files` and `coverage.low_coverage_files` from the updated per-file data.
5. **Log merged files**: Print a summary of how many files had cross-package coverage merged:
   ```
   Cross-package coverage merged:
   - {file_path}: best coverage {line_pct}% (from {pkg_name}, also covered by {other_pkgs})
   ```
   Only print files where merging actually changed a value (i.e., the file appeared in multiple packages with different coverage).

Proceed to Step 3 after merging.

### Step 3: Aggregate and generate HTML report

Using the collected JSON results from all subagents, generate `test-quality-report.html` with all CSS inline and **Chart.js** loaded from CDN for all chart visualizations (treemaps, bar charts, etc.).

#### Report Structure

```
1. LEGEND (always visible at top, collapsible)
   - How to Read This Report section explaining every visual element

2. GLOBAL OVERVIEW
   - Packages analyzed (count + names)
   - Total test count
   - Average line / function coverage (or "N/A" per package if unavailable)
   - Severity distribution (CRITICAL/HIGH/MEDIUM/LOW counts)
   - **Package Size & Coverage bar chart** (all packages side-by-side — see below)
   - **Coverage vs Complexity scatter plot** (all files across all packages — see below)

3. TOP 10 CRITICAL UNCOVERED SECTIONS PER PACKAGE
   - Grouped by package, then ranked list with file, line range, function signature, severity, user impact
   - Each entry is a clickable anchor that jumps to the relevant package section

4. PER-PACKAGE SECTIONS
   For each package:
   a. Package name + language badge
   b. Metrics table (prod files, test files, test cases, prod LOC, test LOC)
   c. Coverage summary (line, function with bars)
   d. **Coverage treemap** (one per package, rendered via Chart.js — see below)
   e. **File coverage heatmap** (grid of files colored by coverage — see below)
   f. **Coverage by test type breakdown** (unit vs integration vs other — see below)
   g. Low-coverage files table
   h. Blind spots list with severity badges
   i. User impact statements
   j. Test design quality (strengths, weaknesses, suggestions)
   k. Complex untested files

5. APPENDIX
   - Full file lists (collapsible)
   - Glossary of terms
   - Methodology notes
```

#### Legend Section (MANDATORY — must be the first section after the title)

The report MUST include a "How to Read This Report" legend section immediately after the title. It must explain:

**1. Report Sections**
- **Global Overview**: High-level summary across all analyzed packages
- **Top 10 Critical Uncovered Sections Per Package**: The most dangerous code with no test coverage per package, ranked by risk
- **Coverage Treemap**: Interactive visualization — larger rectangles = more code, redder = less coverage
- **Package Sections**: Detailed breakdown per package with metrics, coverage, blind spots, and recommendations
- **Appendix**: Methodology and glossary

**2. Metrics Explained**
| Term | Meaning |
|------|---------|
| Production LOC | Lines of non-test source code (excluding blanks and comments) |
| Test LOC | Lines of test source code (excluding blanks and comments) |
| Test Cases | Number of individual test functions/methods detected |
| Line Coverage | % of executable source lines executed during tests |
| Function Coverage | % of functions/methods called at least once during tests |

**3. Coverage Bars**
| Range | Color | Meaning |
|-------|-------|---------|
| < 30% | Red | Critical — most code paths untested |
| 30 – 59% | Orange | Weak — many code paths untested |
| 60 – 79% | Yellow | Moderate — some paths still uncovered |
| >= 80% | Green | Good — most code paths exercised by tests |

**4. Severity Levels**
| Badge | Color | Criteria | Examples |
|-------|-------|----------|----------|
| CRITICAL | Red | Code that, if broken, causes data loss, security issues, or core feature failure | Persistence layers, authentication, financial calculations, critical domain logic |
| HIGH | Orange | Code that, if broken, disrupts major user workflows | User-facing APIs, state management, important business logic |
| MEDIUM | Yellow | Code that, if broken, causes minor user-visible issues | Validation helpers, supporting logic, secondary features |
| LOW | Green | Code that, if broken, has minimal user impact | Formatting utilities, logging, internal helpers |

**5. Blind Spots**
A blind spot is production code identified as **risky due to lack of test coverage**. Each includes:
- **Severity badge**: how critical the gap is (see table above)
- **Description**: what is untested
- **File path**: where the untested code lives (clickable link to GitHub/VS Code)
- **User impact**: what could go wrong for end users if this code breaks
- **Impact example**: a concrete scenario describing the specific bug or UX degradation users would experience

**6. Coverage Treemap**
- Each rectangle represents a **production source file**
- **Size** = lines of code (bigger = more code)
- **Color** = line coverage % (red = low, green = high)
- **Hover** to see file name, LOC, line/function coverage
- **Click** to scroll to the relevant package section
- Use this to quickly identify large, poorly-covered areas of the codebase

**7. Package Size & Coverage Bar Chart**
- Each group of bars represents one package
- **Indigo bar** = production lines of code, **Purple bar** = test lines of code (left Y axis)
- **Coverage bar** = line coverage % (right Y axis, 0-100%)
- Quickly compare package sizes and how well each is tested

**8. Coverage vs Complexity Scatter Plot**
- Each dot represents a **production source file** across all packages
- **X axis** = file size in lines of code (larger = more complex)
- **Y axis** = line coverage % (lower = less tested)
- **Red-tinted zone** (bottom-right) = danger zone: large files with low coverage — prioritize these for testing
- **Color** = package (see legend)

**9. File Coverage Heatmap**
- Grid of colored cells, one per production file in the package
- **Color** = line coverage % (same scale as treemap: red→orange→yellow→green)
- Files sorted worst-first (lowest coverage at top-left)
- **Hover** for full path, LOC, and coverage details

**10. Coverage by Test Type**
- Doughnut chart showing how test cases split across **Unit**, **Integration**, and **Other** types
- **Center number** = total test count
- Table below shows per-type count and coverage % (if available)
- Helps identify if testing is too unit-heavy or lacking integration tests

**11. Test Design Quality**
- **Strengths**: positive testing patterns found in existing tests
- **Weaknesses**: problematic testing patterns or gaps
- **Suggestions**: prioritized recommendations for improving the test suite

**12. Complexity Risks**
Files flagged as both **complex** (high LOC or many conditionals) and **lacking test coverage**. These are the highest-risk files for undetected bugs.

#### Per-Package Coverage Treemap (MANDATORY)

Each package section MUST include an interactive treemap rendered using **Chart.js** with the **chartjs-chart-treemap** plugin. Include the following `<script>` tags once in the `<head>`:
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-chart-treemap@3/dist/chartjs-chart-treemap.min.js"></script>
```

Implementation requirements per package:

1. **Data source**: Use `coverage.per_file` from that package's JSON results
2. **Container**: A `<canvas>` inside a `<div>` with a unique id (e.g., `treemap-{pkg_id}`) and `width: 100%; height: 400px;`
3. **Chart.js config**:
   - `type: 'treemap'`
   - `data.datasets[0].tree`: the per_file array with `{ file, line_pct, function_pct, loc, uncovered_lines }` entries
   - `data.datasets[0].key`: `'loc'` (size each rectangle by lines of code)
   - `data.datasets[0].labels.display`: `true`, `labels.formatter`: show filename (last path component)
   - `data.datasets[0].backgroundColor`: function that maps `line_pct` to color using the coverage color scale
   - `data.datasets[0].borderColor`: `'#fff'`, `borderWidth`: 2
   - `data.datasets[0].spacing`: 2
4. **Colors**: Map `line_pct` to color:
   - 0-29%: `#dc2626` (red)
   - 30-59%: `#ea580c` (orange)
   - 60-79%: `#ca8a04` (yellow)
   - 80-100%: `#16a34a` (green)
   - Use interpolation within each range for granularity
5. **Tooltip**: custom callback showing file name, LOC, line %, function %
6. **Responsive**: set `responsive: true` and `maintainAspectRatio: false` in chart options
7. **Legend**: Include a horizontal color gradient bar below the treemap showing 0% → 100% scale
8. **Skip** the treemap for packages where `coverage.available == false` (show "Coverage unavailable" instead)

#### Package Size & Coverage Bar Chart (MANDATORY — in Global Overview)

A grouped bar chart in the Global Overview comparing all packages side-by-side.

1. **Container**: `<canvas>` inside a `<div>` with id `chart-pkg-size-coverage`, `width: 100%; height: 350px;`
2. **Chart.js config**:
   - `type: 'bar'`
   - **X axis**: one group per package (use short package name, e.g., `BaselineEstimation`)
   - **Datasets** (3 bars per package):
     - `Production LOC` — bar color `#6366f1` (indigo), values from `metrics.production_loc`
     - `Test LOC` — bar color `#8b5cf6` (purple), values from `metrics.test_loc`
     - `Line Coverage %` — bar color mapped by coverage color scale, values from `coverage.line_pct`, plotted on a **secondary Y axis** (right, 0-100%)
   - Left Y axis: LOC scale (label: "Lines of Code")
   - Right Y axis: percentage scale 0-100% (label: "Coverage %")
   - `plugins.legend.display: true`
   - `responsive: true`, `maintainAspectRatio: false`
3. **Tooltip**: show package name, exact LOC, exact coverage %
4. **Skip packages** where coverage is unavailable (show LOC bars only, no coverage bar)

#### Coverage vs Complexity Scatter Plot (MANDATORY — in Global Overview)

A scatter plot showing every production file across all packages, with coverage on Y axis and complexity (LOC) on X axis.

1. **Container**: `<canvas>` inside a `<div>` with id `chart-coverage-complexity`, `width: 100%; height: 400px;`
2. **Data source**: Flatten all `coverage.per_file` entries from all packages. Each point = one file.
3. **Chart.js config**:
   - `type: 'scatter'`
   - **X axis**: `loc` (Lines of Code) — label: "File Size (LOC)"
   - **Y axis**: `line_pct` (0-100%) — label: "Line Coverage %"
   - **One dataset per package**, each with a distinct color from: `['#6366f1', '#ec4899', '#f59e0b', '#10b981', '#3b82f6', '#ef4444', '#8b5cf6']`
   - `pointRadius: 5`, `pointHoverRadius: 8`
   - `plugins.legend.display: true` (shows package names)
   - `responsive: true`, `maintainAspectRatio: false`
4. **Tooltip**: show file name (last path component), package, LOC, line %, function %
5. **Danger zone highlight**: Draw a subtle red-tinted rectangle annotation for the quadrant where `loc > 200` AND `line_pct < 50` (large + poorly covered). Use the Chart.js annotation plugin or a custom `beforeDraw` plugin:
   ```js
   plugins: [{ beforeDraw(chart) {
     const { ctx, scales } = chart;
     const xScale = scales.x, yScale = scales.y;
     ctx.fillStyle = 'rgba(220, 38, 38, 0.05)';
     ctx.fillRect(xScale.getPixelForValue(200), yScale.getPixelForValue(50),
       xScale.getPixelForValue(xScale.max) - xScale.getPixelForValue(200),
       yScale.getPixelForValue(0) - yScale.getPixelForValue(50));
   }}]
   ```

#### File Coverage Heatmap (MANDATORY — per package)

A grid visualization where each cell represents a production file, colored by coverage.

1. **Container**: A `<div>` with id `heatmap-{pkg_id}`, rendered as a CSS grid (not a Chart.js canvas)
2. **Data source**: `coverage.per_file` from that package
3. **Implementation** (pure HTML/CSS — no chart library needed):
   - CSS Grid: `display: grid; grid-template-columns: repeat(auto-fill, minmax(120px, 1fr)); gap: 4px;`
   - Each cell: a `<div>` with:
     - Background color mapped from `line_pct` using the coverage color scale (same as treemap)
     - Text: short filename (last path component), truncated with `text-overflow: ellipsis`
     - Font size: `11px`, color: white for dark backgrounds, dark for light backgrounds
     - `title` attribute: full path, LOC, line %, function %
     - `border-radius: 4px`, `padding: 6px 8px`
     - Min height: `40px`, `cursor: pointer`
   - Sort files: worst coverage first (lowest `line_pct` at top-left)
4. **Color scale**: same as treemap (red <30%, orange <60%, yellow <80%, green >=80%)
5. **Skip** if `coverage.available == false`

#### Coverage by Test Type (MANDATORY — per package)

A doughnut chart showing the distribution of test cases by type (unit vs integration vs other).

1. **Container**: `<canvas>` inside a `<div>` with id `testtype-{pkg_id}`, `width: 300px; height: 300px; margin: 0 auto;`
2. **Data source**: `coverage_by_test_type` from that package's JSON
3. **Chart.js config**:
   - `type: 'doughnut'`
   - **Labels**: `['Unit', 'Integration', 'Other']`
   - **Data**: `[coverage_by_test_type.unit.count, coverage_by_test_type.integration.count, coverage_by_test_type.other.count]`
   - **Colors**: `['#6366f1', '#f59e0b', '#94a3b8']` (indigo, amber, slate)
   - `plugins.legend.position: 'bottom'`
   - `responsive: true`, `maintainAspectRatio: true`
4. **Center text**: Show total test count in the doughnut hole using a custom plugin:
   ```js
   plugins: [{ beforeDraw(chart) {
     const { ctx, width, height } = chart;
     ctx.save(); ctx.font = 'bold 24px system-ui'; ctx.textAlign = 'center';
     ctx.textBaseline = 'middle'; ctx.fillStyle = '#1e293b';
     ctx.fillText(totalTests, width/2, height/2);
     ctx.font = '12px system-ui'; ctx.fillText('tests', width/2, height/2 + 20);
     ctx.restore();
   }}]
   ```
5. **Below the chart**: A small table showing per-type count and coverage % (if available)
6. **Skip** if all counts are 0

#### Top 10 Critical Uncovered Sections Per Package

Render `critical_uncovered_sections` grouped by package, then as a numbered list within each package with:
- **Package header** with package name
- **Rank number** (1-10) in a circle
- **Function signature** in monospace font (the `symbol` field), rendered as a clickable link that opens the file at the specific line in either GitHub (`https://github.com/{org}/{repo}/blob/{branch}/{file}#L{line_start}-L{line_end}`) or VS Code (`vscode://file/{absolute_path}:{line_start}`)
- **File path** with line range (e.g., `CoreEngine.swift:45-92`), also a clickable link (GitHub link as primary `href`, VS Code link as a secondary icon/button)
- **Severity badge**
- **Coverage bar** showing line_pct
- **User impact** statement
- **Clickable**: each entry also links/scrolls to the corresponding file in the package section or treemap

#### Visual Requirements

- Coverage bars: colored progress bars (red <30%, orange <60%, yellow <80%, green >=80%) — two bars per package: line, function
- Severity badges: colored pills with CRITICAL/HIGH/MEDIUM/LOW
- Metrics in card layout
- Tables with alternating row colors
- Collapsible sections for detailed file lists
- Clean, professional design — all CSS inline, only external JS dependencies are Chart.js and chartjs-chart-treemap from CDN
- Mobile-responsive
- **Code links**: All file paths in the report must be clickable. Primary link opens GitHub at the specific line (`https://github.com/{org}/{repo}/blob/{branch}/{file}#L{line}`). Secondary link (small VS Code icon/button) opens VS Code at the line (`vscode://file/{absolute_path}:{line}`). Detect `{org}`, `{repo}`, and `{branch}` from the local git remote and current branch.
- Legend section with a subtle distinct background (`#f1f5f9`) so it's visually separate from report content

#### Color Scheme

```
Background: #f8fafc
Cards: #ffffff
Text: #1e293b
Borders: #e2e8f0
Legend background: #f1f5f9
CRITICAL: #dc2626
HIGH: #ea580c
MEDIUM: #ca8a04
LOW: #16a34a
Coverage bar bg: #e2e8f0
Chart.js tooltip: uses default Chart.js styling
```

### Step 4: Collect all artifacts to output folder

After generating the HTML report:

1. **Create output folder**: `~/Desktop/test-quality-report/`
2. **Save the HTML report** to `~/Desktop/test-quality-report/test-quality-report.html`
3. **Copy original coverage reports** from each package into per-package subfolders:
   - For each package, create `~/Desktop/test-quality-report/coverage/{sanitized-package-name}/`
   - Sanitize package name: replace `/` with `_`, strip leading dots (e.g., `Packages/MyLib` → `Packages_MyLib`)
   - Copy every file/directory listed in `coverage.original_report_paths` into that subfolder, preserving filenames
   - Use `cp -r` for directories (e.g., `htmlcov/`, `.xcresult`)
   - Skip packages where `coverage.original_report_paths` is empty
4. **Save raw JSON results** per package to `~/Desktop/test-quality-report/data/{sanitized-package-name}.json`
5. **Print summary** of all generated files to the user

#### Output folder structure

```
~/Desktop/test-quality-report/
├── test-quality-report.html          # Main HTML report
├── data/
│   ├── Packages_MyLib.json           # Raw analysis JSON per package
│   └── CommonSwift_HWR.json
└── coverage/
    ├── Packages_MyLib/
    │   ├── lcov.info                 # Copied from original location
    │   └── coverage-summary.json
    └── CommonSwift_HWR/
        └── codecov.json
```

6. **Open** `~/Desktop/test-quality-report/test-quality-report.html` in browser after generation.
