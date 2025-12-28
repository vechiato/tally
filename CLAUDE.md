# CLAUDE.md

This file provides mandatory guidance for Claude when working on this codebase.

## Testing Requirements

- **MUST** add tests for new features in the analyzer. Tests go in `tests/test_analyzer.py`.
- **MUST** use the Playwright MCP to test any changes to the HTML report. Generate a report with test data and verify the UI works correctly before committing.

## Development

- **MUST** use `uv run` to run tally during development:
  ```bash
  uv run tally --help
  uv run tally run /path/to/config
  ```
- **MUST NOT** use `python -m tally` or direct Python invocation.

## Releases

- **MUST** use the GitHub workflow for releasing new versions.
- **MUST NOT** create releases manually or tag commits directly.

## Commit Messages

- **MUST** use `Fixes #<issue>` or `Closes #<issue>` syntax when fixing GitHub issues to auto-close them:
  ```
  Fix tooltip display on mobile

  Fixes #42
  ```
- **MUST NOT** commit without referencing the issue when working on a tracked issue.

## Configuration Changes

- **MUST** make backwards compatible changes to `settings.yaml` format. Existing user configs MUST continue to work without modification.
- **MUST** implement automatic migration in `config_loader.py` if a breaking change is unavoidable.
- **MUST** document new configuration options in `config/settings.yaml.example`.
- **MUST** update AGENTS.md (in `src/tally/cli.py`) when adding new user-facing features.

## Error Messages & Diagnostics

- **MUST** make error messages self-descriptive and guide users/agents on what to do next.
- **SHOULD** include specific suggestions in error messages (e.g., "Add: columns:\n  description: \"{field} ...\"").
- **SHOULD** use `tally diag` to diagnose configuration issues. It shows:
  - Config directory and settings file status
  - Data sources with parsed format details (columns, custom captures, templates)
  - Merchant rules (baseline + user rules)
- The tool MUST be self-descriptive enough that users can fix issues without external documentation.

## Project Structure

- `src/tally/` - Main source code
  - `analyzer.py` - Core analysis and HTML report generation
  - `merchant_utils.py` - Merchant normalization and rules
  - `format_parser.py` - CSV format parsing
  - `config_loader.py` - Configuration loading and migration
  - `cli.py` - CLI commands and AGENTS.md template
- `tests/` - Test files
- `docs/` - Marketing website (GitHub Pages)
- `config/` - Example configuration files
