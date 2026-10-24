# 003 Standards and CI

## Goal

Add CI validation and clean up .gitignore.

## Context

This is a Docker Compose infrastructure repo with no application code. CI validates config files rather than running tests.

## Scope

- Add `.github/workflows/ci.yml` that runs `docker compose config` and `nginx -t` on push/PR to main and dev
- Clean up `.gitignore` (remove boilerplate for languages not used in this repo)
- Verify `.env.example` uses placeholder values

## Out of scope

- Lint or format tools (no application code to lint)
- Baseline tests (no app code)

## Acceptance criteria

- CI workflow validates docker-compose.yaml and nginx config on push/PR to main and dev
- `.gitignore` is trimmed to only relevant patterns
- `.env.example` uses only placeholders

## Test plan

- `docker compose config` exits cleanly
- CI YAML is valid

## Notes

- `nginx -t` runs inside a docker container in CI since nginx is not installed on the runner
- `docker compose config` requires the `.env` file, so CI creates a dummy one from `.env.example`
