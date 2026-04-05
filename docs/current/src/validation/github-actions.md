# GitHub Actions Integration

This chapter documents the existing CI/CD configuration and provides patterns for extending it with config validation and formatting checks.

**Source**: `.github/workflows/run-tests.yaml`, `.github/workflows/docs.yml`, `bin/run-tests.sh`

## Existing CI Workflow

The primary CI workflow is `.github/workflows/run-tests.yaml`:

| Setting | Value |
|---|---|
| Trigger | Pushes and PRs to `main` and `dev` branches |
| Runner | `ubuntu-latest` |
| Python version | 3.11 |
| Dependencies | `requirements.txt` |
| Test command | `pytest -v --cov=ft --cov-report=html --cov-report=xml -s tests/` |
| Coverage threshold | >10% |

The workflow installs all dependencies, runs the full test suite with coverage collection, and generates both HTML and XML coverage reports.

## Running Tests Locally

```bash
# Full test suite with coverage
./bin/run-tests.sh

# Single test file
pytest -v -s tests/test_datasets.py

# Single test method
pytest -v -s tests/test_datasets.py::TestDatasets::test_add_dataset
```

The `bin/run-tests.sh` script mirrors the CI configuration. Run it before pushing to catch failures early.

## Adding Config Validation to CI

Axolotl YAML configs and dataset format JSON files live under `ft/config/`. A dedicated workflow validates these files on any PR that modifies them:

```yaml
name: Validate Configs

on:
  pull_request:
    paths:
      - 'ft/config/**'
      - 'data/project_defaults.json'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install pyyaml pydantic

      - name: Validate Axolotl configs
        run: |
          python -c "
          import yaml, json, glob

          # Validate YAML configs
          for f in glob.glob('ft/config/axolotl/training_config/*.yaml'):
              with open(f) as fh:
                  yaml.safe_load(fh)
              print(f'OK: {f}')

          # Validate dataset format JSON configs
          for f in glob.glob('ft/config/axolotl/dataset_formats/*.json'):
              with open(f) as fh:
                  json.load(fh)
              print(f'OK: {f}')

          # Validate project defaults
          with open('data/project_defaults.json') as fh:
              json.load(fh)
          print('OK: data/project_defaults.json')
          "
```

The `paths` filter ensures this workflow only runs when config files change. Any parse error causes the step to fail with a traceback identifying the malformed file.

## Pre-Commit Formatting Check

The project uses `autoflake` and `autopep8` for code formatting. Add a CI step to verify formatting compliance:

```yaml
- name: Check formatting
  run: |
    pip install autoflake autopep8
    autoflake --check --remove-all-unused-imports \
      --ignore-init-module-imports --recursive ft/ tests/ pgs/
    autopep8 --diff --max-line-length 120 \
      --aggressive --aggressive --aggressive \
      --recursive ft/ tests/ pgs/ | head -20
```

`autoflake --check` exits non-zero if any unused imports are found. `autopep8 --diff` prints the diff that would be applied; pipe through `head -20` to keep output concise. If either tool reports issues, the step fails.

## Documentation Deployment

The documentation workflow is `.github/workflows/docs.yml`. It builds the mdbook with D2 diagram support and deploys to GitHub Pages. This workflow is independent of the test CI and triggers on documentation changes. See the [System Overview](../architecture/overview.md) for the full project architecture.

## Extending the CI Pipeline

When adding new validation workflows, follow these conventions:

| Convention | Guideline |
|---|---|
| Path filtering | Use `paths:` to scope workflows to relevant directories |
| Python version | Pin to `3.11` to match production |
| Dependency isolation | Install only what the validation step needs, not the full `requirements.txt` |
| Exit codes | Rely on tool exit codes for pass/fail -- avoid custom success checks |
| Artifact uploads | Use `actions/upload-artifact@v4` for coverage reports or validation logs |
| Branch protection | Configure required status checks on `main` to enforce green CI before merge |
