---
name: test-weaver
description: "Intelligent test suite generation and weaving from requirements, code, and coverage reports"
version: "2.1.0"
author: "OpenClaw Engineering Team"
tags: ["test", "automation", "coverage", "quality", "integration", "weaving"]
category: "quality-assurance"
maintainer: "devops@openclaw.io"
dependencies:
  - "python>=3.9"
  - "pytest>=7.0"
  - "pytest-cov>=4.0"
  - "libcst>=1.0"  # for Python AST manipulation
  - "jq>=1.6"  # JSON processing
  - "git>=2.30"
  - "docker>=20.10"  # optional: for containerized test execution
inputs:
  - "source_code: Path to source directory or specific module"
  - "requirements: Path to requirements document (YAML/Markdown/JSON)"
  - "coverage_report: Existing .coverage or lcov.info file"
  - "existing_tests: Directory of current test suite"
  - "framework: Target test framework (pytest|jest|mocha|googletest)"
  - "strategy: Weaving strategy (coverage-gap|risk-based|spec-compliance)"
outputs:
  - "Generated test files in specified output directory"
  - "Coverage gap analysis report (coverage-gaps.md)"
  - "Weaving manifest (weaving-manifest.json)"
  - "Test execution plan (test-plan.md)"
requires:
  - "docker (optional, for isolated execution)"
  - "git repository with clean working tree"
  - "Write permissions to source and test directories"
exposes:
  - "test-weaver generate"
  - "test-weaver weave"
  - "test-weaver analyze"
  - "test-weaver validate"
  - "test-weaver optimize"
  - "test-weaver rollback"
---

# Test Weaver Skill

Transform requirements and code into comprehensive, intelligently woven test suites with minimal gaps.

## Purpose

Test Weaver automates the creation of thorough test coverage by analyzing source code, requirements specifications, and existing coverage reports to generate missing tests and weave them into coherent suites. It goes beyond simple test generation by understanding test dependencies, execution order, and coverage complementarity.

**Real Use Cases:**

1. **Coverage Gap Filling**: Analyzes `.coverage` or `lcov.info` and generates targeted tests for untested branches, conditions, and exception paths.
2. **Requirements-to-Test Mapping**: Consumes `requirements.yaml` with feature specs and generates compliance test suites ensuring all acceptance criteria are validated.
3. **Legacy Code Testing**: Analyzes un-testested codebases and produces integration tests that exercise complex interactions without requiring full mocking.
4. **Test Suite Weaving**: Merges disjointed test modules into optimized execution plans that reduce redundancy while maintaining independence.
5. **Risk-Based Prioritization**: Generates additional tests for high-risk modules (high complexity, recent changes, critical paths) identified via static analysis.

## Scope

### Commands Handled

```bash
# Generate tests from source code analysis
test-weaver generate --source src/module/ --framework pytest --output tests/generated/

# Generate tests from requirements spec
test-weaver generate --requirements docs/features.yaml --framework jest --tag "login,auth"

# Weave existing test files into optimized suite
test-weaver weave --tests tests/unit/ tests/integration/ --strategy coverage-gap --output tests/woven/

# Analyze coverage and identify gaps
test-weaver analyze --coverage .coverage --source src/ --threshold 85

# Validate woven test suite integrity
test-weaver validate --suite tests/woven/ --dry-run

# Optimize suite execution order
test-weaver optimize --suite tests/ --profile execution-profile.json --output tests/optimized/

# Rollback weaving operation
test-weaver rollback --weaving-id WVR-20240115-ABCD1234 --restore-tests
```

## Detailed Work Process

### 1. Generation Phase

**Input Collection**:
- Parse source code using Tree-sitter or LibCST to build AST and control flow graphs.
- Load coverage reports (pytest-cov, lcov, JaCoCo) to map executed vs. missed lines/branches.
- Read requirements documents (YAML format with `feature:`, `scenarios:`, `acceptance:` keys).
- Scan existing test files to avoid duplication and understand patterns.

**Analysis**:
- Identify untested functions/methods/modules.
- Detect conditional branches without true/false coverage.
- Map requirements to code locations via traceability matrix.
- Classify complexity (cyclomatic) to prioritize test generation.

**Generation**:
- For Python: generate pytest functions with parametrize, pytest fixtures, and mocks using `unittest.mock`.
- For JavaScript/TypeScript: generate Jest test suites with describe/it blocks and proper imports.
- Include edge cases: empty inputs, boundary values, exception raising, async handling.
- Generate test data factories using Factory Boy or test-data-bot patterns.

**Output**:
- Write tests to `tests/generated/` with timestamped filenames: `test_module_generated_20240115_1423.py`.
- Create weaving manifest (`weaving-manifest.json`) linking generated tests to source lines and requirements.
- Generate coverage gap report (`coverage-gaps.md`) with missing line numbers and rationale.

### 2. Weaving Phase

**Dependency Analysis**:
- Parse all test files to extract fixtures, setup/teardown, and shared state.
- Build test dependency graph (nodes=tests, edges=fixture dependencies, shared DB, file I/O).
- Identify tests that can run in parallel (no shared mutable state).

**Strategy Execution**:
- `coverage-gap`: Prioritize tests covering most missing lines; minimize redundant coverage.
- `risk-based`: Order by module complexity × change frequency.
- `spec-compliance`: Group tests by requirements ID for traceability.

**Suite Construction**:
- Order tests to minimize setup/teardown overhead (group by fixture scope).
- Split into shards for parallel CI execution (if `--shards N` provided).
- Generate master runner script that executes in optimal order.

**Output**:
- Optimized test suite in `tests/woven/`.
- Execution plan (`test-plan.md`) with estimated runtime and shard breakdown.
- Dependency graph (`dependencies.dot` or `.json`).

### 3. Validation Phase

- Execute generated tests with `--collect-only` to ensure discoverability by test runner.
- Verify that all imports resolve and fixtures are defined.
- Run subset of critical generated tests to confirm basic validity.
- Check that no generated test exceeds timeout thresholds (e.g., 5s for unit, 30s for integration).
- Ensure coverage report shows improvement (optional `--verify-coverage`).

## Golden Rules

1. **Never modify existing test files**: Generated tests always go to separate directory (`tests/generated/` or `tests/woven/`). Weaving creates new orchestrated suites but preserves originals.
2. **Always maintain test independence**: Generated tests must not rely on execution order or side effects from other tests. Use fixtures or factory patterns.
3. **Include teardown**: Every test that creates resources (DB records, files, network listeners) must have corresponding cleanup.
4. **Be deterministic**: No random seeds without explicit control; avoid time-dependent assertions.
5. **Target 80% coverage minimum** for new modules, but don't sacrifice test quality for quantity.
6. **Preserve existing passing tests**: Weaving never removes or alters original tests; it only adds orchestration layers.
7. **Follow project conventions**: Adhere to existing naming (`test_*.py` vs `*_test.py`), assertion styles (pytest vs unittest), and location patterns.
8. **Document assumptions**: Each generated test file includes header comment linking to requirement ID and coverage gap location.

## Examples

### Example 1: Generate unit tests for uncovered function

```bash
# Command
test-weaver generate \
  --source src/auth/token_manager.py \
  --framework pytest \
  --output tests/generated/auth/ \
  --include-integration false

# Output
INFO: Analyzed src/auth/token_manager.py (342 lines, 18 functions)
INFO: Coverage report: function `refresh_token` has 0% branch coverage on lines 156-178
INFO: Generated test file: tests/generated/auth/test_token_manager_generated_20240115_1442.py
INFO: Created 14 test cases covering 23 uncovered branches
INFO: Weaving manifest: weaving-manifest.json updated with 14 entries

# Generated test snippet (tests/generated/auth/test_token_manager_generated_20240115_1442.py)
import pytest
from unittest.mock import patch, MagicMock
from auth.token_manager import TokenManager

class TestTokenManagerRefreshToken:
    """Generated for coverage gap: refresh_token branch coverage (lines 156-178)"""
    
    @pytest.fixture
    def token_manager(self):
        return TokenManager(secret="test-secret", ttl=3600)
    
    @patch("auth.token_manager.requests.post")
    def test_refresh_token_success(self, mock_post):
        # Test success path with valid refresh token
        ...
    
    @patch("auth.token_manager.requests.post")
    def test_refresh_token_expired_grant(self, mock_post):
        # Test exception when grant is expired
        ...
```

### Example 2: Weave tests into optimized suite

```bash
# Command
test-weaver weave \
  --tests tests/unit/ tests/integration/ \
  --strategy risk-based \
  --shards 4 \
  --output tests/woven/

# Output:
INFO: Scanned 247 test files (1,843 test cases)
INFO: Dependency graph: 1,102 edges, 247 nodes
INFO: Identified 4 optimal shards for parallel execution
INFO: Shard 1 (high-risk): tests/woven/shard-01.yml (512 tests, ~45m)
INFO: Shard 2: tests/woven/shard-02.yml (483 tests, ~38m)
INFO: Shard 3: tests/woven/shard-03.yml (429 tests, ~42m)
INFO: Shard 4 (slow integration): tests/woven/shard-04.yml (419 tests, ~55m)
INFO: Woven suite written to tests/woven/ (4 shard manifest YAMLs)

# Generated shard manifest (tests/woven/shard-01.yml)
shard_id: 1
strategy: risk-based
tests:
  - path: tests/unit/auth/test_token_manager.py::TestTokenManager::test_refresh_token_success
    priority: 10
    estimated_duration: 0.8
    dependencies: ["fixture:db_session"]
  - path: tests/unit/payment/validator_test.py::TestValidator::test_invalid_card_number
    priority: 9
    estimated_duration: 0.3
    dependencies: []
...
```

### Example 3: Analyze coverage and generate gap report

```bash
# Command
test-weaver analyze \
  --coverage coverage-reports/coverage.xml \
  --source src/ \
  --requirements docs/features.yaml \
  --threshold 85 \
  --output reports/

# Output
INFO: Parsed coverage.xml (total lines: 12,453, covered: 9,671, %: 77.6)
INFO: Threshold 85% not met. Missing 1,782 lines across 234 functions.
INFO: Requirements mapping: 18 acceptance criteria lack test validation.
INFO: Generated coverage-gaps.md with prioritized list:

# reports/coverage-gaps.md
## Coverage Gaps - Priority Order

1. **src/payment/processor.py:process_refund (lines 201-245)**
   - Missing branches: exception handling (201-218), edge case (219-230)
   - Requirements: REQ-PAY-004 (Refund processing)
   - Complexity: cyclomatic 12 (high)
   - Action: generate 4 tests covering all branches

2. **src/user/profile.py:update_privacy (lines 87-103)**
   - Missing lines: 95-98 (validation of GDPR fields)
   - Requirements: REQ-USER-011 (Privacy compliance)
   - Complexity: cyclomatic 3
   - Action: generate 1 test with GDPR edge cases

...
```

## Dependencies & Requirements

**System Packages**:
- Python 3.9+ (for test generation engine)
- Node.js 18+ (if generating Jest tests)
- jq (JSON processing)
- git (to detect changed files)

**Python Packages** (`requirements.txt` for test-weaver):
```
pytest>=7.0
pytest-cov>=4.0
libcst>=1.0
PyYAML>=6.0
jinja2>=3.0  # templates for test generation
networkx>=3.0  # dependency graph
click>=8.0  # CLI
```

**Environment Variables**:
```bash
# Optional: override default test directories
export TEST_WEAVER_SOURCE_DIR="src/"
export TEST_WEAVER_TEST_DIR="tests/"
export TEST_WEAVER_GENERATED_DIR="tests/generated/"
export TEST_WEAVER_WOVEN_DIR="tests/woven/"

# CI integration
export TEST_WEAVER_CI_MODE="true"  # skips interactive prompts
export TEST_WEAVER_MAX_TESTS_PER_MODULE="20"  # limit generation
export TEST_WEAVER_TIMEOUT_PER_TEST="5"  # seconds

# Coverage thresholds
export TEST_WEAVER_COVERAGE_THRESHOLD="85"
```

## Verification Steps

After running any test-weaver command, verify:

1. **Generated tests are syntactically valid**:
   ```bash
   python -m py_compile tests/generated/*.py
   # or for JS:
   npx jshint tests/generated/*.js
   ```

2. **Tests are discoverable by runner**:
   ```bash
   pytest --collect-only tests/generated/ | grep "test_"
   # should list generated tests without errors
   ```

3. **Coverage improves** (if `--verify-coverage`):
   ```bash
   pytest tests/generated/ --cov=src/module --cov-report=json
   # inspect coverage.json to confirm targeted lines now covered
   ```

4. **Weaved suite executes without dependency errors**:
   ```bash
   pytest tests/woven/shard-01.yml -v  # or use generated runner script
   # should run all tests in shard without import/fixture errors
   ```

5. **Manifest integrity**:
   ```bash
   jq . weaving-manifest.json  # should be valid JSON
   jq -e '.[] | select(.source_line == null)' weaving-manifest.json && echo "FAIL" || echo "PASS"
   # all entries must have source_line and requirement_id
   ```

## Troubleshooting

**Symptom**: Generated tests fail import errors.
- **Cause**: Test framework not in project dependencies.
- **Fix**: Install required framework or specify `--framework none` and adjust templates.

**Symptom**: Weaving produces circular dependency warnings.
- **Cause**: Two tests share fixtures that depend on each other.
- **Fix**: Refactor fixtures to use session-scoped or function-scoped appropriately; use `--strategy coverage-gap` to minimize fixture sharing.

**Symptom**: Coverage doesn’t improve after adding generated tests.
- **Cause**: Generated tests target wrong source paths or exclude patterns in coverage config.
- **Fix**: Ensure `--source` matches import paths; check `.coveragerc` `omit` patterns.

**Symptom**: Test execution extremely slow after weaving.
- **Cause**: Woven suite includes many integration tests that should be separate.
- **Fix**: Use `--shards` and run separately; generate only unit tests with `--include-integration false`.

**Symptom**: Rollback fails with "weaving-id not found".
- **Cause**: Provided ID does not match any entry in weaver’s log.
- **Fix**: Check `weaving-manifest.json` for valid IDs; use `test-weaver list-backups` to see available rollback points.

## Rollback Commands

Test Weaver maintains rollback points before every `generate` and `weave` operation.

```bash
# List available rollback points
test-weaver list-backups
# Output:
# WVR-20240115-ABCD1234: weave 2024-01-15 14:23 UTC (target: tests/woven/)
# WVR-20240115-EFGH5678: generate 2024-01-15 10:11 UTC (target: tests/generated/auth/)

# Rollback a weaving operation (removes woven suite, restores original test selection)
test-weaver rollback --weaving-id WVR-20240115-ABCD1234

# Rollback last operation (latest)
test-weaver rollback --latest

# Rollback but keep generated tests (only remove weaving manifests)
test-weaver rollback --weaving-id WVR-20240115-ABCD1234 --keep-generated

# Full restore: removes both generated and woven tests, reverts git working tree
# WARNING: destructive
test-weaver rollback --weaving-id WVR-20240115-ABCD1234 --hard --git-rollback

# Verify rollback state (dry-run)
test-weaver rollback --weaving-id WVR-20240115-ABCD1234 --dry-run
```

Rollback implementation:
1. Copies of original test directories stored in `.test-weaver-backups/WVR-<id>/`.
2. Git integration: commits pre-operation changes to temporary branch `test-weaver/WVR-<id>`; `--git-rollback` resets to that commit.
3. Weaving manifest maintains `backup_path` field pointing to backup location.

## Performance & Scaling

- **Generation**: ~50-200 tests per minute depending on source complexity.
- **Weaving**: O(V+E) on test dependency graph; 1,000 tests completes in ~5s.
- **Memory**: ~200MB for 10,000 test dependency graph.

Use `--shards N` for CI parallelism; use `--dry-run` to validate without heavy computation.
```