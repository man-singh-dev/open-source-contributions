# Open Source Contributions

My open-source pull requests: what the problem was, how I found the root cause,
how I fixed and tested it, and what I learned. Each contribution has its own
folder with a full write-up and the interview questions it prepares me for.

## Contributions

| # | Project | PR | Issue | Area | Status |
|---|---------|----|-------|------|--------|
| 1 | [Kaneo](https://github.com/usekaneo/kaneo) | [PR #____](https://github.com/usekaneo/kaneo/pull/____) | [#2148](https://github.com/usekaneo/kaneo/issues/2148) | React, browser security, testing | Open |

### 1. Kaneo #2148: Gitea Connect crashes on plain-HTTP deployments

`crypto.randomUUID()` only exists in secure contexts, so self-hosted installs on
`http://<LAN IP>` crashed when opening Gitea integration settings. I replaced it
with a counter where uniqueness was all that mattered, and with
`crypto.getRandomValues()` where real randomness was needed, then added
regression tests for both.

- [Write-up](contributions/kaneo-2148-randomuuid-http/README.md)
- [Interview questions](contributions/kaneo-2148-randomuuid-http/interview-qa.md)

**Skills:** Web security (secure contexts), React (`useMemo`, render errors),
TanStack Query cache keys, Vitest + Testing Library, test-first bug fixing,
git (fork workflow, line endings, hooks), open-source contribution process.

## Adding a new contribution

Copy [`TEMPLATE.md`](TEMPLATE.md) into `contributions/<project>-<issue>-<short-name>/README.md`,
fill it in, and add a row to the table above.
