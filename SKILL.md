---
name: Code Seer
slug: code-seer
version: 1.2.0
description: Predictive code analysis and refactoring guidance based on architectural trends and evolution patterns
author: SMOUJBOT Team
tags:
  - refactoring
  - prediction
  - architecture
  - technical-debt
  - code-quality
  - maintenance
type: analysis
entrypoint: code-seer
depends:
  - python>=3.9
  - tree-sitter
  - networkx
  - scikit-learn
  - pandas
  - rich
  - gitpython
  -SQLite (for caching)
permissions:
  - read:files
  - analyze:code
  - suggest:refactor
environment:
  CODESEER_MODEL: default
  CODESEER_CACHE_TTL: "86400"
  CODESEER_VERBOSE: "false"
  CODESEER_MIN_CONFIDENCE: "0.7"
---

# Code Seer

Predictive code analysis engine that anticipates how your codebase will need to evolve, identifies refactoring opportunities before they become bottlenecks, and provides data-driven architectural recommendations.

## Purpose

### Real Use Cases

1. **Preemptive Refactoring** - Identify code that will become problematic in 3-6 months based on current change patterns and complexity growth
2. **Architecture Drift Detection** - Detect when code diverges from intended architectural patterns (e.g., Clean Architecture, hexagonal) before technical debt accumulates
3. **Dependency Evolution Forecasting** - Predict which modules will become tangled and suggest modularization before coupling becomes unmanageable
4. **API Stability Analysis** - Determine which interfaces are stable vs volatile based on change history and usage patterns
5. **Team Onboarding Intelligence** - Highlight areas of code that frequently change (hotspots) and suggest documentation improvements

## Scope

### Commands

```bash
# Predict code hotspots for next 90 days
code-seer predict-hotspots --path ./src --days 90 --output json

# Analyze architectural drift against Clean Architecture
code-seer check-architecture --pattern clean --boundaries-dir ./docs/architecture

# Generate refactoring roadmap with priority and effort estimates
code-seer roadmap --path . --weeks 12 --format markdown

# Forecast coupling growth and suggest module boundaries
code-seer coupling-forecast --threshold 0.8 --suggest-split

# Identify volatile APIs (interfaces with >3 breaking changes/month)
code-seer api-stability --path ./packages --min-stability-score 0.7

# Detect God Objects and Suggest Splits
code-seer detect-god-objects --complexity-threshold 50 --output-diff
```

## Detailed Work Process

### 1. Codebase Ingestion & Historical Analysis

```
Step 1.1: Git History Mining (30-60 min for medium projects)
  - Extract commit frequency per file/module (last 90 days)
  - Calculate churn metrics: added/deleted lines per commit
  - Identify co-change patterns (files changed together >5 times)
  
  Command: git log --since="90 days ago" --name-only --pretty=format: | sort | uniq -c | sort -nr

Step 1.2: Static Code Analysis
  - Parse AST using tree-sitter for all target languages
  - Calculate cyclomatic complexity, cognitive complexity, lines of code
  - Identify architectural layers (if pattern provided)
  - Extract dependency graph (imports, includes, requires)

Step 1.3: Machine Learning Prediction
  - Load pre-trained model (trained on 10k+ open source repos)
  - Input features: churn, complexity, coupling, team ownership, test coverage
  - Predict: probability of modification in next N days, refactor urgency score
```

### 2. Specific Refactoring Detection

```
Step 2.1: God Object Detection
  Conditions triggering detection:
    - Class with >150 LOC AND >10 methods AND >15 fields
    - Class with >20 direct dependencies (imports/inherits)
    - Class with coupling between 2+ architectural layers
  
  Suggested actions:
    - Extract Class (Extract Methods/Fields to new types)
    - Delegate responsibilities to service objects
    - Introduce interfaces to reduce coupling

Step 2.2: Shotgun Surgery Pattern
  Detection: Multiple unrelated modules modify same file (>3 PRs/week modifying same file)
  Suggestion: Extract Strategy Pattern, use dependency injection

Step 2.3: Feature Envy Detection
  Metrics: Method uses data from another class >70% of its operations
  Suggestion: Move method to the class it's "envious" of
```

### 3. Output Generation

- Primary: Structured JSON with scores and specific line numbers
- Secondary: Human-readable markdown with code snippets showing before/after
- Optional: PR description template ready to copy-paste

## Golden Rules Specific to Code Seer

1. **Never** suggest refactoring without confidence >= `MIN_CONFIDENCE` (default 0.7)
2. **Always** provide concrete line numbers and 3-5 line diff snippets
3. **Prioritize** suggestions by: (impact_score * probability) / estimated_effort_hours
4. **Respect** existing tests - never suggest refactoring that would break >5% test coverage
5. **Validate** all predictions against git history (backtest 30-day accuracy)
6. **Cache** results for 24h to avoid re-analyzing unchanged files
7. **Never** suggest breaking changes to public APIs marked as v1.0+ without migration guide
8. **Always** check for duplicated patterns before suggesting new abstractions

## Examples

### Example 1: Predict Hotspots

**User Input:**
```bash
code-seer predict-hotspots --path ./src --days 90 --format json
```

**Output:**
```json
{
  "hotspots": [
    {
      "file": "src/auth/middleware.ts",
      "prediction": {
        "modification_probability": 0.94,
        "estimated_churn_next_90d": 320,
        "refactor_urgency": "critical",
        "reason": "8 commits in last 30d, cyclomatic complexity 28, 12 dependencies"
      },
      "suggested_refactor": {
        "type": "extract_strategy",
        "description": "Split authentication strategies (OAuth, JWT, Session) into separate classes",
        "estimated_effort_hours": 8,
        "risk": "low"
      }
    }
  ]
}
```

### Example 2: Detect God Objects with Diff

**User Input:**
```bash
code-seer detect-god-objects --complexity-threshold 40 --output-diff
```

**Output:**
```diff
--- a/src/core/UserService.ts
+++ b/src/core/UserService.ts (suggested split)
@@ -1,5 +1,10 @@
+// SUGGESTION: Extract UserValidator and UserNotifier
+// Rationale: Class has 214 LOC, 18 methods, and handles 3 concerns
+
 class UserService {
-  // 214 lines of mixed validation, notification, persistence logic
+  private validator: UserValidator;
+  private notifier: UserNotifier;
   private repository: UserRepository;
 }
```

**Additional file generated:** `refactor-plan/user-service-split-20240115.md` with complete migration steps and test preservation strategy.

### Example 3: Architecture Drift

**User Input:**
```bash
code-seer check-architecture --pattern hexagonal --domain src/domain --ports src/ports --adapters src/adapters --report html
```

**Output:**
```
Architecture Drift Report (Hexagonal)
=====================================

VIOLATIONS FOUND: 3

1. Domain layer depends on Infrastructure (HIGH)
   File: src/domain/models/Order.ts:45
   Imports: import {DatabaseConnection} from '../../infrastructure/db'
   Fix: Move DatabaseConnection to Ports, inject via constructor

2. Adapters directly access Domain entities (MEDIUM)
   Files: src/adapters/http/OrderController.ts:12,23
   Issue: Direct property access instead of using domain methods
   Fix: Use Order.confirm() instead of order.status='confirmed'

3. Missing dependency rule enforcement (INFO)
   Ports layer imports concrete adapter
   Action: Add dependency-check CI step
```

## Dependencies and Requirements

**Required:**
- `tree-sitter` (parsing for 15+ languages)
- `gitpython` (git history analysis)
- `networkx` + `scikit-learn` (prediction models)
- `sqlite3` (result caching)

**Optional (enhanced predictions):**
- `pandas` (statistical analysis)
- `radon` (complexity metrics for Python)
- `eslint`/`typescript` (for JS/TS codebases)

**Installation:**
```bash
pip install -r requirements-code-seer.txt
code-seer init --cache-dir ~/.cache/code-seer --model-dir ~/.local/share/code-seer/models
```

## Verification Steps

```bash
# 1. Verify installation and dependencies
code-seer --version && code-seer health

# Expected output:
# Code Seer v1.2.0
# Cache: healthy (/home/user/.cache/code-seer)
# Model: loaded (architecture-predictor-v3.pkl)
# Tree-sitter: 15 languages available

# 2. Run dry-run to see what would be analyzed
code-seer predict-hotspots --path ./test-projects/sample --dry-run

# Expected: "Would analyze 42 files, 2,847 LOC, 6 months of git history"

# 3. Execute with confidence threshold
code-seer predict-hotspots --path . --min-confidence 0.8

# 4. Validate predictions manually (spot check 2-3 suggestions)
code-seer validate --prediction-file predictions.json --check-accuracy

# Expected: "Backtest accuracy: 87% (predictions matched actual changes)"
```

## Environment Variables

```bash
export CODESEER_MODEL=architecture-predictor-v3  # Default: latest
export CODESEER_CACHE_TTL=86400                  # Cache TTL in seconds (default: 1 day)
export CODESEER_VERBOSE=true                     # Enable debug logging
export CODESEER_MIN_CONFIDENCE=0.65              # Minimum prediction confidence to show
export CODESEER_GIT_BIN=/usr/bin/git             # Override git binary path
export CODESEER_MAX_FILES=10000                  # Safety limit for analysis
export CODESEER_EXCLUDE="node_modules,venv,.git,build"
```

## Troubleshooting Common Issues

### Issue 1: "No git history found"
**Cause:** Running on repo with <30 days of history or shallow clone
**Fix:**
```bash
git fetch --unshallow 2>/dev/null || git fetch --depth=10000
code-seer reset-cache && code-seer predict-hotspots --path .
```

### Issue 2: "MemoryError on large codebase"
**Cause:** Insufficient memory for full AST parsing
**Fix:**
```bash
code-seer predict-hotspots --path . --max-files 5000 --batch-size 100
# Or slice by subdirectory
find src -name "*.ts" -type f | head -n 2000 > filelist.txt
code-seer predict-hotspots --file-list filelist.txt
```

### Issue 3: "Model loading failed"
**Cause:** Missing or corrupted model file
**Fix:**
```bash
code-seer init --download-model architecture-predictor-v3
# Or reset to default
rm -rf ~/.local/share/code-seer/models
code-seer init
```

### Issue 4: "False positive: Suggesting refactor for generated code"
**Cause:** Code Seer analyzing auto-generated files
**Fix:** Add patterns to ignore:
```bash
code-seer config set ignore_patterns '["*.pb.go", "*_generated.*", "migrations/*"]'
# Or use .codeseerignore file (similar to .gitignore)
echo "dist/" >> .codeseerignore
echo "*.min.js" >> .codeseerignore
```

## Rollback Commands

### Undo a Suggested Refactor (if applied manually)
```bash
# Restore from git before refactor branch was merged
git checkout main
git log --oneline --grep="refactor:" | head -1 | cut -d' ' -f1 | xargs git revert --no-edit

# Or use Code Seer to generate rollback diff
code-seer rollback --refactor-id "extract-strategy-20240115" --format patch > rollback.patch
patch -p1 < rollback.patch
```

### Disable Predictions for Specific File/Directory
```bash
# Add permanent exclusion
code-seer config add exclude_path ./legacy/unsupported-module

# Or temporary skip with flag
code-seer predict-hotspots --path . --skip-pattern "legacy/*"
```

### Restore Original Git State (if analysis caused unintended changes)
```bash
# Safety: Code Seer never modifies files, only suggests
# But if you applied suggestions and want to revert:
git reset --hard HEAD
git clean -fd

# If you committed already:
git revert HEAD
```

### Clear Cache and Re-analyze with New Settings
```bash
code-seer cache clear --all
code-seer config set min_confidence 0.6
code-seer predict-hotspots --path .
```

## Performance Notes

- Initial analysis: 1-5 minutes for 10k LOC codebase
- Cache hit subsequent runs: <1 second
- Prediction accuracy: ~85% when backtested against 6-month future changes
- Best results: Run weekly, integrate with CI/CD to track drift over time

```