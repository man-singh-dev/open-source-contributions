# Kaneo #2148: Gitea Connect crashes on plain-HTTP deployments

| | |
|---|---|
| **Project** | [Kaneo](https://github.com/usekaneo/kaneo), an open-source, self-hostable project-management tool |
| **Issue** | [#2148](https://github.com/usekaneo/kaneo/issues/2148) (labels: `bug`, `priority:high`, `ready-for-contribution`) |
| **Pull request** | [#____](https://github.com/usekaneo/kaneo/pull/____) |
| **Branch** | `fix/gitea-random-uuid-http` |
| **Status** | Open |
| **Date** | 2026-10-29 |
| **Tech** | React 19, TypeScript, TanStack Query, TanStack Router, Vitest (vite-plus), Testing Library, pnpm monorepo, Husky, commitlint |
| **Size** | 4 files, ~45 lines (2 source files, 2 test files) |

---

## 1. The problem

Kaneo can be self-hosted. One user ran it on their local network over plain
HTTP (for example `http://192.168.1.10`). In a project's
**Settings → Integrations**, clicking **Connect** next to **Gitea** replaced the
whole page with:

> Something went wrong!
> crypto.randomUUID is not a function

The form never appeared, so Gitea integration was unusable on that kind of
deployment. It reproduced in Chrome and in a Firefox-based browser.

**Reproduce:** serve Kaneo over `http://` on a non-localhost hostname or IP →
open a project's settings → Integrations → click Connect next to Gitea.

---

## 2. Root cause

Three facts combine into the crash.

### a) `crypto.randomUUID()` only exists in secure contexts

Browsers divide pages into **secure contexts** and non-secure ones. A page is a
secure context when it's served over `https://`, or from a local address such
as `http://localhost`, `http://127.0.0.1` or `http://*.localhost`. Those are
trusted because traffic never leaves the machine.

`http://192.168.1.10` and `http://myserver.lan` are **not** secure contexts. On
those pages the browser simply doesn't define `crypto.randomUUID`:

```js
window.isSecureContext   // false
typeof crypto.randomUUID // "undefined"
crypto.randomUUID()      // TypeError: crypto.randomUUID is not a function
```

`crypto.getRandomValues()` is a notable exception: it **is** available in
non-secure contexts.

### b) The picker called it during render

`apps/web/src/components/project/gitea-integration/gitea-repository-picker.tsx`:

```tsx
const credentials = useMemo(
  () => ({ accessToken, cacheId: crypto.randomUUID() }),   // line 39
  [accessToken],
);
```

`useMemo` runs its callback **during render**. An error thrown during render
propagates up to the nearest error boundary. Here that's the router's error
component, which rendered "Something went wrong!" in place of the page.

### c) The picker is mounted even while it's closed

`GiteaSetupPanel` always renders `<GiteaRepositoryPicker open={browsing} … />`.
`open` only controls whether the dialog is visible and whether fetching is
enabled (`canFetch`). The component, and its `useMemo`, still run as soon as
the setup panel appears. So the crash happened right after clicking Connect,
before the user typed anything.

### Why the developers didn't notice

Developers run the app on `http://localhost`, which counts as a secure context.
The bug only appears on self-hosted plain-HTTP deployments, which is exactly the
environment developers don't test in.

### History

- Introduced in commit `71bd1983` (2026-09-21, *"fix: enforce security
  boundaries and bound integration workloads"*), first released in **v2.26.0**.
- Moved into the current file in `3f157107` (*"feat(web): rebuild settings
  pages"*, v2.42.0).
- Reported against **v2.48.0**.

---

## 3. Understanding *why* `cacheId` exists (the key insight)

Before changing anything I had to understand what the UUID was for. The test
file names answer it: *"Gitea repository credential isolation"*, *"hides
privileged cached repositories immediately when a token changes"*, *"cannot
display a late response from the previous token"*.

The query that lists repositories looks like this:

```tsx
useQuery({
  queryKey: ["gitea-repositories", session?.user.id, projectId, baseUrl, credentials.cacheId],
  queryFn: () => listGiteaRepositories({ projectId, baseUrl, accessToken: credentials.accessToken }),
  enabled: canFetch,
  gcTime: 0,
  placeholderData: () => undefined,
});
```

**Scenario it protects against:** you paste an admin token and see private
repositories. Then you switch to a limited token. If the query key were the
same, TanStack Query would show the **cached admin results** under the limited
token. That's a data leak.

**Why not just put `accessToken` in the key?** Query keys are visible in React
Query DevTools, end up in cache maps, and can be persisted or logged. Secrets
shouldn't live there.

**So `cacheId` is an opaque stand-in for the token.** Its only requirement:

> A value that is **new every time the token changes** and never repeats within
> the page.

It does **not** need to be random or unguessable: it never leaves the browser,
and it isn't a secret. That realisation is what makes the simplest fix correct.

---

## 4. Options I considered

| Option | Verdict |
|---|---|
| `crypto.randomUUID?.() ?? Math.random().toString(36)` fallback | Works, but keeps two code paths for something that doesn't need randomness |
| Only mount the picker when `open` is true | Avoids the crash at mount, but it still crashes when the user clicks "Browse" |
| Show "HTTPS required" | The issue prefers working on HTTP, and nothing here truly needs a secure context |
| Third-party `uuid` package | Extra dependency for a non-problem |
| `useId()` | ❌ Stable for the component's lifetime, so it would **not** change when the token changes, which breaks credential isolation |
| **Module-level counter** | ✅ Simplest, works on any origin, keeps the isolation guarantee |

For the second file (workspace slugs), randomness **does** matter, so the right
tool there is `crypto.getRandomValues()`.

---

## 5. The fix

### a) Gitea picker: counter

```diff
 type GiteaRepository = ListGiteaRepositoriesResponse["repositories"][number];

+// A fresh id per token keeps repositories cached for one token out of another.
+// A counter instead of crypto.randomUUID(), which is missing on plain-HTTP origins.
+let credentialCacheId = 0;
+
 ...
   const credentials = useMemo(
-    () => ({ accessToken, cacheId: crypto.randomUUID() }),
+    () => ({ accessToken, cacheId: ++credentialCacheId }),
     [accessToken],
   );
```

Why it's enough:
- `useMemo` depends on `[accessToken]` → a new token reruns it → the counter increments.
- The counter lives at **module level**, so it's shared by every picker instance and only
  ever goes up. Two different tokens can never get the same id while the page is open.
- The token itself still stays out of the query key.

### b) Workspace slug: same bug, different need

While searching for other `crypto.randomUUID` uses in browser code, I found
`apps/web/src/lib/utils/create-workspace-slug.ts`:

```diff
+// getRandomValues, unlike randomUUID, also works on plain-HTTP origins.
 function createRandomSuffix(): string {
-  return crypto.randomUUID().replace(/-/g, "").slice(0, RANDOM_SUFFIX_LENGTH);
+  const bytes = crypto.getRandomValues(
+    new Uint8Array(RANDOM_SUFFIX_LENGTH / 2),
+  );
+  return Array.from(bytes, (byte) => byte.toString(16).padStart(2, "0")).join(
+    "",
+  );
 }
```

**Why randomness matters here and a counter would be wrong:** workspace slugs
(e.g. `acme-3f9a1c07b2de`) must be unique across **all users** on the server.
The browser only knows the current user's own workspaces. If the slug collides
with someone else's, the server rejects it and `create-workspace.ts` retries
with a new suffix, up to 5 times. A counter would generate `acme-1`, `acme-2`,
… in every user's browser, so collisions would be constant. Random suffixes
make collisions very unlikely.

- `RANDOM_SUFFIX_LENGTH` is 12 hex characters = 6 bytes = **48 bits** of randomness.
- Each byte becomes 2 hex chars: `toString(16)` → `padStart(2, "0")` so `0x0a` → `"0a"`, not `"a"`.
- `RANDOM_SUFFIX_LENGTH / 2` is used instead of a hardcoded `6`, so changing the length later still works.

When does this path run? Only when the base slug is taken or reserved
(`api`, `auth`, `dashboard`, … are reserved because they clash with app routes).

### c) Not affected

`apps/api` also uses `randomUUID`, but that's Node.js server code. Node always
provides it; the secure-context rule is browser-only.

---

## 6. How I tested it (test-first: red → green)

### Regression test 1: picker without `randomUUID`

```tsx
it("loads repositories on plain-HTTP origins without crypto.randomUUID", async () => {
  const original = Object.getOwnPropertyDescriptor(crypto, "randomUUID");
  Object.defineProperty(crypto, "randomUUID", { value: undefined, configurable: true });
  try {
    m.list.mockResolvedValueOnce(result("lan-repo"));
    show();
    await screen.findByText("owner/lan-repo");
  } finally {
    if (original) Object.defineProperty(crypto, "randomUUID", original);
    else delete (crypto as { randomUUID?: unknown }).randomUUID;
  }
});
```

- **Simulates a plain-HTTP browser** by removing `randomUUID`.
- `Object.defineProperty` instead of plain assignment, because `crypto` properties
  may not be writable.
- Saves the **full property descriptor** and restores it in `finally`, so
  other tests are unaffected even if this one fails.
- **Before the fix:** failed with `TypeError: crypto.randomUUID is not a function`
  at `gitea-repository-picker.tsx:39`, the same error as the issue.
- **After the fix:** passes.

### Regression test 2: slug without `randomUUID`

Calls `createUniqueWorkspaceSlug("api", [])` and expects `/^api-[0-9a-f]{12}$/`.

**Trap avoided:** the function returns early (no random suffix) when the base
slug is free. A test like `createUniqueWorkspaceSlug("my-team", [])` would
pass even **before** the fix and prove nothing. `"api"` is a reserved slug,
so it always takes the random-suffix path.

### Existing tests

The 4 existing credential-isolation tests pass unchanged. They cover token
change, a late response from the old token, and a session change.

### Other checks

- The pre-commit hook ran `vp check` (format + lint) over all 4,520 files in the repo: passed.
- commitlint checked the commit message format: passed after a fix (see below).

---

## 7. Problems I hit along the way

| Problem | Cause | Fix |
|---|---|---|
| Test command hung for 5+ minutes | Output piped through `Select-Object -Last` (buffers everything until the process exits), plus an earlier attempt in watch mode | Ran `vp test run <file>` directly in a terminal |
| `git diff main` showed `:` and waited | Git's pager (`less`) | `q` to quit, or `git --no-pager diff` |
| New test file missing from `git diff` | Untracked files aren't shown by `git diff` | `git status` / `git diff HEAD --stat` |
| Pre-commit hook: **"formatting issues in 4,516 files"** | Git for Windows `core.autocrlf=true` checked every file out with CRLF line endings; the formatter expects LF | `git stash -u` → `git config core.autocrlf false` → `git rm -r --cached .` → `git reset --hard` → `git stash pop` |
| commitlint: `body-max-line-length` | Commit body was one line over 100 chars | Split the body into several `-m` paragraphs |
| AI review bot failed | Bot-side error, unrelated to the code | Retried once with `/agentic_review` |

---

## 8. Open-source process

- The issue was labeled `ready-for-contribution`. An eligibility bot only
  accepts PRs that link such an issue (`Fixes #2148`); other PRs are
  converted to drafts.
- Commented on the issue with my proposed approach before opening the PR.
- Followed Conventional Commits: `fix(web): avoid crypto.randomUUID on plain-HTTP origins`.
- Used the repo's PR template (*What changed? / Related issue / How did you check it?*).
- Kaneo's AI policy: AI may help write code, but the contributor must understand
  it, and PR text must be in their own voice.

---

## 9. Review feedback

_(fill in after review)_

---

## 10. What I learned

1. **Some browser APIs only exist on HTTPS.** Examples: `crypto.randomUUID`,
   `crypto.subtle`, `navigator.clipboard`, service workers, geolocation.
   `localhost` hides this during development.
2. **Ask what a value is *for* before replacing it.** "Unique within the page" and
   "random across all users" need different tools.
3. **A test must fail before the fix**, otherwise it may not test what you think
   (the slug early-return trap).
4. **A cache key is part of security.** Stale cache data across credentials is a leak.
5. **Search for the same bug elsewhere** once you've found one instance.
6. **Read the repo's rules first:** CONTRIBUTING, PR template, commit format,
   AI policy, eligibility.
7. **Windows + git line endings** can break a whole repo's formatting checks.

## Follow-up ideas (not in this PR)

- Add `* text=auto eol=lf` to `.gitattributes` so Windows contributors don't hit the CRLF problem.
- Add a lint rule (e.g. `no-restricted-properties`) banning `crypto.randomUUID` in `apps/web`.
- Mount the picker only when it's open (separate improvement, less work on first render).

---

## How I worked

I used Claude Code to help implement the fix and run commands. I reviewed every
change, checked the diff myself, understood why each line is needed, and wrote
the PR description in my own words.
