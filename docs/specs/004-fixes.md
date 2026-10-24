# 004 Fixes

## Goal

Remove real IPs, domains, and hostnames from tracked files. Report gitleaks findings.

## Context

The README and ngrok/resolv.conf contained real IP addresses and a real domain name. No bugs were noted from baseline tests (Phase 4 was skipped for this infrastructure repo).

## Scope

- Replace all real IPs with `192.168.x.x` placeholders in README.md and ngrok/resolv.conf
- Replace real domain with `example.com` in README.md
- Run gitleaks and report findings

## Out of scope

- History rewriting for gitleaks findings (owner decides)

## Acceptance criteria

- No real IPs or domains in tracked files
- gitleaks findings documented

## Test plan

- Grep for real IP patterns and domain names across tracked files

## Notes

gitleaks found 2 leaks in git history (not in current files):

1. **NGROK_AUTHTOKEN** in `.env` at commit `873f973` (Add nginx reverse proxy and ngrok tunnel). The `.env` was deleted in a later commit and added to `.gitignore`. The ngrok token is from a now-unused ngrok account.

2. **Node-RED credentialSecret** in `node-red/data/.config.runtime.json` at commit `5e1e8fa` (Add Docker Compose with Mosquitto and Node-RED). The file was removed when `node-red/*` was added to `.gitignore`.

Both secrets exist only in history, not in the current working tree. Recommend rotating the ngrok token if the account is still active.
