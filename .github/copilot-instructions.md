# GitHub Backup - AI Coding Instructions

## Project Overview

This is a Python CLI tool for comprehensive GitHub data backup. The architecture follows a single-module design with clear separation of concerns across functional areas.

## Core Architecture

### Main Entry Points
- **`bin/github-backup`**: CLI entry point that orchestrates the backup workflow
- **`github_backup/github_backup.py`**: Single module containing all core functionality (~1400+ lines)
- **`github_backup/__init__.py`**: Version tracking only

### Data Flow Pattern
1. **Parse & Authenticate** → `parse_args()` → `get_auth()` → `get_authenticated_user()`
2. **Discover** → `retrieve_repositories()` → `filter_repositories()`  
3. **Backup** → `backup_repositories()` + `backup_account()`

### GitHub API Integration
- Uses `retrieve_data_gen()` for paginated API calls with automatic rate limiting
- Template-based URL construction: `"https://{host}/repos/{owner}/{name}/issues"`
- Built-in retry logic for 502 errors and incomplete reads
- Supports both classic tokens (`-t`) and fine-grained tokens (`-f`)

## Key Development Patterns

### Authentication Flexibility
```python
# Supports multiple auth methods in get_auth():
# - Fine-grained tokens (github_pat_...) 
# - Classic tokens with x-oauth-basic
# - Basic username/password
# - OSX Keychain integration
# - GitHub App authentication (--as-app)
```

### Incremental Backup Strategy
- **Time-based**: `--incremental` uses API `since` parameter with last backup timestamp
- **File-based**: `--incremental-by-files` compares filesystem modification times
- State stored in `{output_dir}/last_update` file

### Git Repository Handling
- Uses `logging_subprocess()` wrapper for all git operations
- Supports both regular clones and bare/mirror clones (`--bare` → `git clone --mirror`)
- SSH vs HTTPS preference via `--prefer-ssh` flag
- LFS support with `git lfs fetch --all --prune`

### Output Directory Structure
```
{output_dir}/
├── repositories/{repo_name}/repository/     # Git clones
├── starred/{owner}/{repo_name}/             # Starred repos
├── gists/{gist_id}/                         # User gists  
├── account/{starred,followers,following}.json
└── {repo}/issues/{number}.json              # Per-repo data
```

## Development Workflows

### Testing & Linting
```bash
# No unit tests exist - this is acknowledged in README
pip install flake8
flake8 --ignore=E501,E203,W503  # Same as CI
```

### Docker Development
```bash
docker run --rm -v /path/to/backup:/data --name github-backup \
  ghcr.io/josegonzalez/python-github-backup -o /data $OPTIONS $USER
```

### Release Process
- Automated via GitHub Actions (`automatic-release.yml`, `tagged-release.yml`)
- Version bumping in `github_backup/__init__.py`
- Docker image publishing to ghcr.io

## Critical Implementation Details

### Rate Limiting Strategy
- Automatic throttling based on `x-ratelimit-remaining` header
- Custom throttling via `--throttle-limit` and `--throttle-pause`
- Exponential backoff for 403 rate limit responses

### Error Handling Philosophy
- Graceful degradation for missing data (404s logged but don't block)
- Blocking errors (403 auth failures) exit entirely
- Incomplete reads get 3 retry attempts with 5-second delays

### File I/O Patterns
- Atomic writes via `.temp` files then `os.rename()`
- UTF-8 encoding with `codecs.open()` for JSON files
- JSON formatting: `ensure_ascii=False, sort_keys=True, indent=4`

## Common Gotchas

1. **`--all` doesn't include everything**: Missing private repos, forks, starred repos, LFS, gists
2. **`--bare` is actually `--mirror`**: Uses `git clone --mirror`, not `git clone --bare`
3. **Starred gists**: Stored in same directory as user gists, not separately
4. **Incremental risks**: Failed runs can cause missing data in subsequent incremental backups
5. **Authentication scope**: Fine-grained tokens need specific repository and user permissions

## Extension Points

When adding new backup types, follow the pattern:
1. Add CLI argument in `parse_args()`
2. Create `backup_*()` function following existing patterns
3. Call from `backup_repositories()` or `backup_account()`
4. Use `retrieve_data()` for API calls and `mkdir_p()` for directories
5. Follow atomic file writing pattern with `.temp` files
