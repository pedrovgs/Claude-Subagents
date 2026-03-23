# Generate Build-Safe Code Mutations

You are a mutation testing generator. Your job is to produce **build-validated code mutations** for automated mutation testing.

## Arguments

- `$ARGUMENTS` — parse the following named arguments:
  - `packages` (required): comma-separated list of packages/modules to analyze
  - `output` (optional): path to output JSON file
  - `max_per_package` (optional): maximum mutations per package
  - `parallelism` (optional): number of parallel build validations (default: 4)
  - `help`: shows how to use this command

## Step 0: Parse Arguments and Interactive Prompts

1. Parse `$ARGUMENTS` into named parameters. Extract `packages`, `output`, `types`, `max_per_package`, `parallelism`.
2. If `packages` is missing, stop and ask the user which packages to analyze.
3. If `max_per_package` is NOT provided, prompt the user interactively:
   > "How many mutations per package do you want to generate? (default: 100)"
   If the user provides no value or says default, use **100**.
4. If `output` is NOT provided, default to `~/Desktop/mutations.json`.
5. If `parallelism` is NOT provided, default to `4`.
6. If `help` show documentaiton about how to use this command.

## Step 1: Detect Commit Hash

Run `git rev-parse HEAD` to capture the current commit hash. This will be embedded in the output.

## Step 2: Identify Source Files

For each package in the `packages` list:
1. Locate the package directory. Use the project structure from AGENTS.md:
   - Check `Packages/<name>`, `CommonSwift/<name>`, `Core/<name>`, `iOS/<name>`, `ts-packages/<name>`, `CrossplatformWeb/<name>`, and other known paths.
   - If ambiguous, search for the package using glob patterns.
2. Collect all source files (`.swift`, `.ts`, `.tsx`, `.js`, `.jsx`, `.kt`, `.java` depending on context).
3. Exclude test files, generated files, and vendored code.

## Step 3: AST-Based Mutation Generation

For each source file, perform **AST-level analysis** to generate candidate mutations. Do NOT use regex.

### How to Perform AST Analysis

Read each source file and parse its structure by understanding:
- Function/method bodies
- Expressions and operators
- Control flow (if/else, guard, switch, for, while)
- Return statements
- Boolean expressions
- Numeric literals and constants
- Optional/nullability patterns

### Mutation Types to Generate

#### Standard Mutations
- **arithmetic_operator**: `+` to `-`, `-` to `+`, `*` to `/`, `/` to `*`, `%` to `*`
- **logical_operator**: `&&` to `||`, `||` to `&&`
- **comparison**: `==` to `!=`, `!=` to `==`, `<` to `<=`, `<=` to `<`, `>` to `>=`, `>=` to `>`
- **boolean_flip**: `true` to `false`, `false` to `true`
- **constant_perturbation**: `n` to `n+1`, `n` to `n-1`, `0` to `1`, `1` to `0`

#### Advanced Mutations
- **condition_negation**: wrap condition in `!()` or remove existing `!`
- **statement_removal**: remove a statement that has no compile-time dependency (e.g., a void function call, a logging statement)
- **return_mutation**: change return value (e.g., return opposite boolean, return 0 instead of computed value)
- **loop_boundary**: off-by-one in loop bounds (`<` to `<=`, `i = 0` to `i = 1`)
- **optional_handling**: change `.some` to `.none`, remove `?` nil-coalescing defaults
- **control_flow**: add early `return` at start of function, swap if/else branches

If `types` filter is provided, only generate mutations matching those types.

### Mutation Generation Rules
- Each mutation must be a **single, atomic change** at a specific file:line:column
- Record the `original` code snippet and the `mutated` replacement
- Assign a unique `id` to each mutation (format: `<package>-<file>-<line>-<type>-<index>`)
- Classify as `standard` or `advanced` category
- Cap at `max_per_package` mutations per package (prioritize diversity of types)

## Step 4: Build Validation

For EACH candidate mutation:

1. **Apply in isolation**: Create a temporary copy or use git stash/patch approach:
   - Copy the original file
   - Apply the single mutation to the file
   - Build ONLY the affected package/module
   - Restore the original file immediately after

2. **Build command selection**:
   - For Swift packages in `CommonSwift/` or `Packages/`: use `swift build` within the package directory
   - For iOS targets: use `xcodebuild` with the appropriate scheme (build-only, no tests)
   - For TypeScript packages: use `tsc --noEmit` or the package's build script
   - For Kotlin: use `./gradlew :module:compileKotlin`

3. **Result**:
   - Build succeeds -> mark `build_validated: true`, keep mutation
   - Build fails -> discard mutation entirely

### Performance Optimizations
- Run up to `parallelism` validations concurrently
- Use incremental builds where the build system supports it
- If a file has multiple mutations, batch-test them (apply one at a time, but reuse the build cache between attempts)
- Track which files have been validated for this commit to avoid redundant work

## Step 5: Caching

Implement file-based caching:
- Cache directory: `.claude/mutation-cache/`
- Cache key: `<commit-hash>-<file-path-hash>-<mutation-id>`
- Before validating a mutation, check if it was already validated for the same commit+file content
- Store cache entries as small JSON files
- Skip regeneration if file is unchanged (compare file hash) and mutation was already validated

## Step 6: Generate Output JSON

Write the final JSON to the output path:

```json
{
  "commit": "<commit-hash>",
  "generated_at": "<ISO-8601 timestamp>",
  "packages": ["pkg1", "pkg2"],
  "total_candidates": <number of candidates before validation>,
  "total_validated": <number that passed build>,
  "mutations": [
    {
      "id": "core-MathUtils-42-arithmetic_operator-0",
      "package": "core",
      "file": "path/to/file.swift",
      "location": {
        "line": 42,
        "column": 10
      },
      "original": "a + b",
      "mutated": "a - b",
      "type": "arithmetic_operator",
      "category": "standard",
      "build_validated": true
    }
  ]
}
```

## Step 7: Print Summary

After writing the JSON file, print a human-readable summary to the console:

```
Mutation Generation Summary
===========================
Commit: <hash>

Total candidates analyzed: <N>
Total validated mutations: <N> (<percentage>% survival rate)

Mutations per package:
  - <package>: <count>

Mutation types:
  - <type>: <count>

Top affected files:
  - <file>: <count>
  - <file>: <count>
  - <file>: <count>

Output written to: <path>
```

## Safety Requirements

- NEVER leave modified files in the working directory. Always restore originals.
- NEVER commit or push any changes.
- Use `git stash` or file copy/restore to ensure clean state.
- Remove all the stashed changes before you consider the work done
- If the process is interrupted, ensure cleanup happens (restore files).
- All mutations must be applied in complete isolation from each other.

## Execution Strategy

Use the Agent tool to parallelize work:
1. Spawn one agent per package for mutation candidate generation (read-only exploration)
2. For build validation, work sequentially within each package but parallelize across packages
3. Aggregate results from all agents into the final JSON

## Important Notes

- This is a **code generation and build validation** task, not just research
- You MUST actually apply mutations and run builds to validate them
- You MUST write the output JSON file
- You MUST print the summary
- Prioritize correctness over speed: a smaller set of guaranteed-valid mutations is better than a large set with false positives
