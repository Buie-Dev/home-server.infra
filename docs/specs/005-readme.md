# 005 README

## Goal

Rewrite the README to match the cleanup style rules while preserving the architecture documentation.

## Context

The repo already has a detailed README with architecture diagrams and definitions. It needs to be reformatted to the standard sections and writing style (plain first person, no dashes as punctuation, no semicolons/colons in prose, no filler, CI badge).

## Scope

- Rewrite README.md with standard sections in order
- Add CI badge
- Keep the architecture diagram and definitions (they are valuable)
- Written in plain first person

## Out of scope

- Screenshots (infrastructure repo, no UI)

## Acceptance criteria

- README has all required sections in correct order
- CI badge at the top
- No dashes as punctuation, no semicolons, no colons in prose
- No filler words or emoji
- No real IPs or domains

## Test plan

- Review README for accuracy and style compliance

## Notes

- Sections: Title and summary, Why I built it, How it fits, Stack, Running it locally, Configuration table, Tests, How I work on it
- The architecture and definitions sections are kept as additional content after the standard sections
