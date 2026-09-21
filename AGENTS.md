# Repository Guidelines

## Project Structure & Module Organization
`dirsearch.py` is the main CLI entrypoint. Core code lives in `lib/`: `lib/connection/` handles HTTP/DNS, `lib/controller/` drives scans and sessions, `lib/core/` stores settings and option state, `lib/report/` contains output handlers, and `lib/view/` formats terminal output. Bundled wordlists and data files live in `db/`. Tests are under `tests/`, with static fixtures in `tests/static/`. Packaging files live in `pyproject.toml`, `setup.py`, `requirements/`, and `pyinstaller/`.

## Adding a New CLI Flag (e.g. HTTP version control)
CLI parsing uses `optparse`, not `argparse`. New options are defined in `lib/parse/cmdline.py` in the matching `OptionGroup` (request-line options go in the "Request Settings" group). A flag shows in default `--help` only if its long name is in `COMMON_HELP_OPTIONS`; otherwise it appears only under `--help-all`. Adding a name to that set without defining the option raises `RuntimeError` at startup (checked by `tests/parse/test_cmdline.py`).

Flag plumbing chain: `cmdline.py` option -> config.ini default in the matching section -> `merge_config()` in `lib/core/options.py` (pattern: `opt.x = opt.x or config.safe_get("section", "key", default)`) -> consumed in `lib/connection/requester.py` via the shared `options` dict from `lib/core/data.py`. Start-time validation/combination rules (e.g. engine forcing, option conflicts) live in `parse_options()` in `lib/core/options.py`.

## HTTP Version Flag (`--http-version`)
`--http-version` accepts `0.9`, `1.0`, `1.1` (default `1.1`). Allowed values are `HTTP_VERSIONS` in `lib/core/settings.py`.

- `1.1` is the default and works in every backend.
- `1.0` and `0.9` force the sync threaded engine in `parse_options()` and error out if `-a/--async` is explicit. `0.9` additionally rejects proxies/replay-proxy, non-GET methods, and request bodies.
- `1.0` is sent via `_HTTPVersionControlledRequestMixin.putrequest()` in `lib/connection/requester.py`, which sets http.client's `_http_vsn`/`_http_vsn_str` per connection; urllib3 (both lines) reads those attributes when building the request line, so keep the mixin on all four `PathPreserving*Connection` classes.
- `0.9` uses a raw socket exchange (`Requester._request_http09()`): the request is `GET /path\r\n` with no headers and the response is body-only until close. `_parse_http09_response()` leniently parses modern `HTTP/x.y` replies so status codes and headers still reach filters/reports. Results are `HTTP09Response` (`lib/connection/response.py`).
- The native Rust backend hardcodes `HTTP/1.1` (`native/src/raw_client.rs`) and the framing parser `native/src/raw_http.rs` is HTTP/1.1-only; non-1.1 versions are rejected in `get_native_request_backend_error()` (`lib/core/request_backend.py`).
- HTTPS targets with `0.9` are rejected in `Controller.set_target()`.
- httpx has no HTTP/1.0 mode; the async engine therefore only supports `1.1` (see the engine forcing above).

Coverage locations: `tests/parse/test_cmdline.py`, `tests/core/test_options_config_merge.py`, `tests/core/test_options.py`, `tests/connection/test_requester.py` (includes a raw `HTTP09TCPServer` harness and `request_versions` assertions), and `tests/core/test_request_backend.py` for backend validation.

## Build, Test, and Development Commands
- `python3 dirsearch.py -u https://example.com -w tests/static/wordlist.txt -q`: quick local CLI smoke test.
- `python -m unittest discover -s tests -t .`: canonical unit/integration test runner used by CI.
- `python3 -m unittest tests.connection.test_requester tests.connection.test_dns tests.core.test_scanner`: focused regression pass.
- `python3 -m pip install .`: validate packaged install and console entrypoints.
- `docker compose -f - build dirsearch` with `build.network: host`: verify Docker release images, matching the GitHub workflow.
- `pyinstaller --clean pyinstaller/dirsearch.spec`: build the standalone binary using the checked-in spec.

## Coding Style & Naming Conventions
Use 4-space indentation and keep Python code straightforward and modular. Prefer descriptive `snake_case` for functions and variables, `PascalCase` for classes, and small helpers for exception-heavy logic. Keep module boundaries clean: networking belongs in `lib/connection`, report logic in `lib/report`, and CLI/config parsing in `lib/parse` or `lib/core`. Follow the existing flake8 rules in `pyproject.toml`.

## Testing Guidelines
Tests use `unittest`. Add new coverage under `tests/` with filenames like `test_requester.py` and methods named `test_*`. When changing request, packaging, or report behavior, add message-level or artifact-level assertions rather than only smoke checks. For compatibility-sensitive changes, prefer Docker validation on supported Python versions.

## Docker Release Validation
When changing `Dockerfile`, Docker workflows, release packaging, dependencies, or stack defaults, build through `docker compose` with host networking instead of plain `docker build`, because CI relies on `build.network: host` for dependency resolution. Validate every release stack, not just the default one: `threaded`, `async`, and `native-rust`.

Use this pattern for each stack, replacing `STACK` and image tag as needed:

```sh
docker compose -f - build dirsearch <<'YAML'
services:
  dirsearch:
    image: dirsearch:test-STACK
    build:
      context: .
      dockerfile: Dockerfile
      network: host
      args:
        DIRSEARCH_STACK: STACK
YAML
```

After each image builds, run smoke tests for `--version`, `--help`, and a minimal scan:

```sh
docker run --rm dirsearch:test-STACK --version
docker run --rm dirsearch:test-STACK --help
docker run --rm \
  -v "$PWD/tests/static/wordlist.txt:/tmp/wordlist.txt:ro" \
  dirsearch:test-STACK \
  -u https://example.com -w /tmp/wordlist.txt -e html -q
```

## Native Backend Benchmark Results
Phase 5 native request-backend benchmark results are summarized in `docs/native-backend-benchmarks.md`. Do not commit raw benchmark JSON or temporary benchmark runner scripts unless they are explicitly needed for a reproducible regression investigation.

When reporting benchmark results, lead with medians and include best samples only as secondary context. Distinguish direct HTTP client numbers from full `dirsearch` contention numbers because the full scan includes fuzzer, callbacks, filters, process overhead, and scheduler effects. Treat runtime worker count and HTTP in-flight concurrency as separate knobs: runtime workers should follow CPU count, while HTTP concurrency may be higher for I/O-bound scans.

## Commit & Pull Request Guidelines
Recent history favors short imperative commits such as `Fix async SSL classification and add tests` or `Use PyInstaller spec in GitHub workflows`. Keep commits scoped to one change. PRs should explain the user-visible effect, link the issue when applicable, and list the commands you ran. Update docs or workflow files when changing CLI flags, packaging, or bundled artifacts.

Do not use automation-specific prefixes such as `codex/` in branch names or `[codex]` in PR titles. Use plain descriptive branch names and PR titles that fit the project history.

## Security & Configuration Tips
Do not commit secrets, session artifacts, local virtualenv files, or cloud benchmark artifacts. DigitalOcean tokens must stay in environment variables such as `DIGITALOCEAN_ACCESS_TOKEN`; never write them into scripts, result files, or docs.

Never include private infrastructure details in commits, docs, PR bodies, PR comments, issues, or release notes. This includes internal hostnames, machine names, SSH targets, provider account details, regions, IP addresses, private paths, runner names, or operational topology. When validation used private infrastructure, describe only the generic platform and toolchain, such as "validated on Windows amd64 with Python 3.13 and MSVC Rust."

If you change dependencies, keep `requirements.txt`, `requirements/runtime.txt`, and packaging metadata aligned. If you add runtime files or imports, verify both `pyinstaller/dirsearch.spec` and GitHub Actions workflows still bundle them.
