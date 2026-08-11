## 9. Testing and Code Quality

### Types of Testing

Unit: isolated unit; no external dependencies; maximum speed.
Integration: component interaction with real dependencies.
Functional / E2E: black-box from the user's perspective.

### Code Quality

- One function = one logical operation (SRP)
- Pure functions: result depends only on arguments, no side-effects
- Nesting: no more than 3 levels
- Cyclomatic complexity (mccabe): ≤ 8 paths
- Meaningful names

Tools: `flake8`, `pylint`, `black`, `isort`, `mypy`, `bandit` (security), `pytest`, `coverage.py`.

---
