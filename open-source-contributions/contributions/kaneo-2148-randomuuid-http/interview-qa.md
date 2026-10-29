# Interview questions: Kaneo #2148

Practice answering out loud **before** reading each answer.
⭐ = very likely to be asked.

**Contents**
1. [Explaining the PR](#1-explaining-the-pr)
2. [Browser security & secure contexts](#2-browser-security--secure-contexts)
3. [Randomness, UUIDs & IDs](#3-randomness-uuids--ids)
4. [React](#4-react)
5. [TanStack Query & caching](#5-tanstack-query--caching)
6. [Testing](#6-testing)
7. [Git & tooling](#7-git--tooling)
8. [Open source & teamwork](#8-open-source--teamwork)
9. [Tricky follow-ups](#9-tricky-follow-ups)
10. [Behavioural (STAR)](#10-behavioural-star)

---

## 1. Explaining the PR

**⭐ Q: Tell me about this contribution in 30 seconds.**
Kaneo is a self-hostable project-management app. Users running it over plain
HTTP on their LAN got a full-page crash when connecting Gitea. The cause was
`crypto.randomUUID()`, which browsers only provide on HTTPS or localhost. It was
being used only to make a cache key unique per access token, so I replaced it
with a simple counter. I found the same bug in workspace slug generation, where
real randomness is needed, and used `crypto.getRandomValues()` there. I wrote
regression tests that failed before the fix and pass after.

**⭐ Q: What was the root cause, precisely?**
`crypto.randomUUID` is `undefined` in non-secure contexts. The picker called it
inside `useMemo` during render, so it threw a `TypeError` and the router's error
boundary replaced the page. Because the setup panel always mounts the picker
(even while closed), it crashed immediately after clicking Connect.

**Q: How did you confirm the root cause?**
The error message pointed straight at the API. I read the picker and found the
only call. Then I wrote a test that removes `crypto.randomUUID` and renders the
picker. It failed with the identical error at line 39.

**Q: Why did nobody catch this earlier?**
`localhost` counts as a secure context, so it works in development. It only
breaks on non-localhost HTTP, which developers rarely use.

**Q: Which files did you change?**
`gitea-repository-picker.tsx` and its test, plus `create-workspace-slug.ts` and a
new `create-workspace-slug.test.ts`.

**Q: How big was the change? Why so small?**
About 45 lines including tests. A small fix is easier to review and less risky.
The goal was to fix the bug, not refactor the component.

**Q: What would you do differently?**
Reproduce it in a real browser over a LAN IP as well, not only with a unit test,
and include a screenshot in the PR.

---

## 2. Browser security & secure contexts

**⭐ Q: What is a secure context?**
A page whose origin the browser trusts not to be tampered with in transit:
`https://`, `wss://`, and local addresses such as `http://localhost`,
`http://127.0.0.1`, `http://[::1]` and `http://*.localhost`. Also `file://` in most
browsers. Powerful APIs are restricted to secure contexts.

**Q: Why is `localhost` treated as secure over HTTP?**
Traffic to loopback never leaves the machine, so nobody on the network can
intercept or modify it. It also keeps development easy.

**Q: Why isn't `http://192.168.1.10` secure if it's "only" on my home network?**
The browser can't know your network is trustworthy. Anyone on the same Wi-Fi
could intercept or inject traffic (man-in-the-middle).

**⭐ Q: How do you check for a secure context in code?**
`window.isSecureContext` (or `self.isSecureContext` in workers). Or
feature-detect: `typeof crypto.randomUUID === "function"`.

**Q: Name other APIs that require a secure context.**
`crypto.subtle` (Web Crypto), `crypto.randomUUID`, `navigator.clipboard`,
service workers, Geolocation, `getUserMedia` (camera/mic), Web Bluetooth/USB,
Push/Notifications, Payment Request, WebAuthn.

**Q: Is `crypto.getRandomValues()` restricted?**
No, it works in non-secure contexts too. That's why it was safe for the slug fix.

**Q: What's the difference between `crypto.getRandomValues` and `crypto.subtle`?**
`getRandomValues` just fills a typed array with random bytes. `subtle` is the
full Web Crypto API (hashing, encryption, signing, keys), and it requires a
secure context.

**Q: What's feature detection and why prefer it to browser sniffing?**
Checking whether an API exists (`if ("randomUUID" in crypto)`) instead of
checking the browser name or version. It's accurate for the actual environment,
including a secure-context difference in the *same* browser.

**Q: Should the app force users onto HTTPS instead?**
HTTPS is better practice, but self-hosted LAN installs often don't have
certificates. The app shouldn't crash on a feature that doesn't need a secure
context. Where a feature truly needs HTTPS, show a clear message rather than
throwing.

---

## 3. Randomness, UUIDs & IDs

**Q: What is a UUID v4?**
A 128-bit identifier written as 36 characters (`xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`).
122 bits are random; the other 6 encode the version (4) and variant.

**⭐ Q: Why was a counter acceptable for the picker but not for slugs?**
The picker's id only needs to be unique **within one page**, and change when the
token changes. It never leaves the browser and isn't secret. Slugs must be
unique **across all users on the server**, and each browser only knows its own
workspaces. A counter would generate the same `acme-1` for everyone, while random
suffixes make collisions unlikely.

**Q: Why not `Math.random()`?**
It isn't cryptographically secure and implementations can be predictable. For
the picker, randomness isn't needed at all. For slugs, `getRandomValues` is
available and better. `Math.random().toString(36)` also gives variable-length
output.

**Q: How much randomness does the slug suffix have?**
12 hex chars = 6 bytes = 48 bits ≈ 2.8 × 10¹⁴ possibilities. Same as the old
code, which took the first 12 hex characters of a UUID.

**Q: When would two random suffixes probably collide (birthday paradox)?**
Around √(2⁴⁸) = 2²⁴ ≈ 16.7 million suffixes for the **same base name**. And the
server rejects duplicates anyway; the client then retries (up to 5 times).

**⭐ Q: Explain `byte.toString(16).padStart(2, "0")`.**
`toString(16)` converts a byte (0–255) to hex. Values below 16 produce one
character (`10` → `"a"`), so `padStart(2, "0")` makes it `"0a"`. Every byte then
becomes exactly 2 characters and the suffix is always 12.

**Q: Why `new Uint8Array(RANDOM_SUFFIX_LENGTH / 2)`?**
Each byte is 2 hex characters, so 12 characters need 6 bytes. Using the constant
keeps it correct if the length changes.

**Q: Is there a limit on `getRandomValues`?**
Yes, 65,536 bytes per call; larger arrays throw `QuotaExceededError`. It also
only accepts integer typed arrays (not `Float32Array`).

**Q: Could the counter overflow?**
Practically no. JavaScript numbers are exact integers up to 2⁵³ − 1
(`Number.MAX_SAFE_INTEGER`), far beyond any number of token changes in one page.

**Q: Does the counter reset? Is that a problem?**
It resets on page reload. That's fine because the TanStack Query cache also lives
in memory and is gone after a reload, so old ids can't clash with old entries.

---

## 4. React

**⭐ Q: What does `useMemo` do?**
It caches the result of a calculation between renders and only recalculates
when a dependency changes (compared with `Object.is`).

**⭐ Q: Is `useMemo` a guarantee?**
No. React's docs say it's a performance optimisation and React may discard the
cache (e.g. during development or for offscreen components). Here, if React
recomputed it, the counter would just give a new id. The only effect would be one
extra fetch, never wrong data, so correctness doesn't depend on memo stability.
That was true of the old UUID code as well.

**Q: What happens in React Strict Mode?**
In development, React calls the `useMemo` callback twice to surface impure
code. The counter may skip a number (1 → 3). Harmless, since only uniqueness matters.

**Q: Isn't incrementing a module variable inside `useMemo` a side effect?**
Technically yes; it's impure. It's acceptable here because the only observable
effect is getting a fresh unique number, which can't cause incorrect behaviour.
A purist alternative is `useState` with a lazy initialiser plus a reset when the
token changes, or a `useRef` updated when the token changes, at the cost of more code.

**⭐ Q: Why not `useId()`?**
`useId` returns the same id for the whole lifetime of a component instance, so
it wouldn't change when the token changes. The query key would stay the same,
and repositories cached under the old token could reappear. It's also meant
for accessibility attributes (`aria-describedby`), not cache keys.

**Q: Why does an error during render crash the whole page?**
React unmounts the tree up to the nearest **error boundary**. Here that was the
router-level error component, which replaced the whole page with "Something
went wrong!".

**Q: What do error boundaries catch, and what don't they catch?**
They catch errors in rendering, lifecycle methods and constructors of their
children. They don't catch errors in event handlers, async code (`setTimeout`,
promises), SSR, or in the boundary itself.

**Q: Why was the picker mounted when closed?**
The parent renders it unconditionally and passes `open`. That keeps dialog
state and animation simple, but its hooks still run on every render.

**Q: Would `{browsing && <GiteaRepositoryPicker … />}` have fixed it?**
Only partly. The page would load, but it would still crash when the user opened
the picker. It also resets the picker's internal state each time it closes.

**Q: Module-level state in React: any pitfalls?**
It's shared across all instances and persists across renders and unmounts.
With server-side rendering it's shared across **all requests and users** on the
server, which can leak data. Kaneo's web app is client-rendered and the counter
holds no user data, so it's safe here.

---

## 5. TanStack Query & caching

**⭐ Q: What's a query key?**
An array that identifies cached data. Same key → same cache entry. A changed key
→ a new entry and a new fetch.

**⭐ Q: Why does `cacheId` matter for security?**
Without it, switching from an admin token to a limited token would keep the same
key and could show the admin's private repositories. A new id per token
guarantees a fresh cache entry.

**Q: Why not put the token directly in the key?**
Keys are visible in React Query DevTools, live in the cache map, and can be
persisted or logged. Secrets shouldn't be stored there. `cacheId` is an opaque stand-in.

**Q: What does `gcTime: 0` do?**
Unused cache entries are garbage-collected immediately, so results for an old
token don't linger in memory.

**Q: What does `placeholderData: () => undefined` do here?**
It prevents showing the *previous key's* data while the new key loads (which
`keepPreviousData` would do). Otherwise a token change would briefly display
the old token's repositories.

**Q: What does `enabled: canFetch` do?**
The query won't run until it's true: dialog open, user signed in, URL and token
filled in. It stops the network request, but **not** the code that builds the
key. That's why the crash still happened with `enabled: false`.

**Q: What's the "late response" problem?**
A slow request for token A finishes after you switched to token B. Because the
response is stored under A's key, it can't show up under B's key. One of the
existing tests checks this.

---

## 6. Testing

**⭐ Q: What is a regression test?**
A test that reproduces a specific bug, so if the bug ever comes back the test fails.

**⭐ Q: Why write the test before the fix?**
To prove it reproduces the bug. If it already passes before the fix, it isn't
testing the bug (red → green).

**⭐ Q: How did you simulate a non-secure browser in a test?**
By removing `crypto.randomUUID` with `Object.defineProperty(crypto, "randomUUID",
{ value: undefined, configurable: true })`, then restoring the original
descriptor in `finally`.

**Q: Why `Object.defineProperty` and not `crypto.randomUUID = undefined`?**
Properties on `crypto` may be non-writable or defined on the prototype, so plain
assignment can silently fail or throw in strict mode. `defineProperty` sets an
own property reliably.

**Q: Why save the whole descriptor with `getOwnPropertyDescriptor`?**
To restore exactly what was there (value, writable, enumerable, configurable).
If `randomUUID` was inherited rather than own, there's no own descriptor, so the
test deletes the own property to uncover the inherited one again.

**Q: Why `try/finally`?**
If an assertion fails, `finally` still restores `randomUUID`, so later tests
aren't broken by leaked global state.

**Q: Could you use `vi.stubGlobal` or `vi.spyOn` instead?**
`vi.spyOn(crypto, "randomUUID").mockImplementation(() => { throw … })` simulates
a throw, but not "undefined". `vi.stubGlobal("crypto", {...})` replaces the
whole object, which affects `getRandomValues` too. `defineProperty` on just one
property is the most precise.

**⭐ Q: Explain the slug-test trap.**
`createUniqueWorkspaceSlug` returns the plain base slug when it's free and never
calls `createRandomSuffix`. A test with an unused name passes even with the bug.
The test must force the suffix path, by passing a name that's already taken or
reserved like `"api"`.

**Q: What do `findByText` and `queryByText` differ in?**
`findBy*` is async and waits (retries) until the element appears, or throws.
`queryBy*` is sync and returns `null` if not found, which is useful for asserting
absence. `getBy*` is sync and throws if not found.

**Q: Why mock `listGiteaRepositories`?**
Unit tests shouldn't call a real Gitea server. Mocking makes tests fast and
deterministic, and lets you control success, error and slow responses.

**Q: What does `vi.hoisted` do?**
`vi.mock` calls are hoisted to the top of the file, above imports.
`vi.hoisted` creates variables that are also hoisted, so mock factories can use them.

**Q: Unit test vs end-to-end test for this bug?**
The unit test is fast and pins the root cause. An E2E test (Playwright served
on a non-localhost HTTP origin) would prove the whole user flow, but it's slower
and harder to set up.

---

## 7. Git & tooling

**⭐ Q: Describe your fork-and-PR workflow.**
Fork → clone my fork → create a branch → commit → push the branch to my fork →
open a PR into `usekaneo/kaneo:main` → respond to review by pushing more commits.

**Q: Why a separate branch?**
It keeps `main` clean, one PR per branch, and makes it easy to update the PR or
start other work.

**⭐ Q: The pre-commit hook said 4,516 files were badly formatted. What happened?**
Git for Windows' `core.autocrlf=true` converts LF to CRLF on checkout. The
formatter expects LF, so every file looked wrong. I stashed my work, disabled
`autocrlf` for the repo, re-checked out all files with LF, and restored my changes.

**Q: LF vs CRLF?**
Line endings: LF (`\n`) on Linux/macOS, CRLF (`\r\n`) on Windows.

**Q: How should a repo prevent line-ending problems?**
A `.gitattributes` with `* text=auto eol=lf` normalises line endings for
everyone, regardless of their local git settings.

**Q: What do `git stash -u` and `git stash pop` do?**
`stash` saves uncommitted changes and cleans the working tree; `-u` includes
untracked files. `pop` reapplies them and removes the stash entry.

**Q: Why didn't my new file appear in `git diff`?**
`git diff` shows unstaged changes to tracked files. New (untracked) or already
staged files don't appear. Use `git status`, `git diff --staged` or `git diff HEAD`.

**Q: What is Husky?**
A tool that installs git hooks from the repo. Kaneo runs `pnpm lint` before
every commit (`pre-commit`) and commitlint on the message (`commit-msg`).

**⭐ Q: What are Conventional Commits?**
A message format `type(scope): description`, e.g.
`fix(web): avoid crypto.randomUUID on plain-HTTP origins`. Types like `feat`,
`fix`, `docs` and `chore` let tools generate changelogs and version numbers.
Kaneo uses semantic-release, so `fix` produces a patch release.

**Q: Why did commitlint reject your first message?**
The body had one line longer than 100 characters (`body-max-line-length`).

**Q: Why stage files by name rather than `git add .`?**
It ensures only intended files go into the commit, not logs, build output or
accidental changes.

**Q: What is a monorepo and pnpm workspace?**
One repo containing several packages/apps (`apps/web`, `apps/api`,
`packages/*`). pnpm workspaces link them and share one lockfile;
`--filter` runs commands for specific packages.

---

## 8. Open source & teamwork

**Q: How did you pick this issue?**
It was labeled `ready-for-contribution` and `priority:high`, had a detailed
report with a suspected cause, and was small enough to do well.

**Q: What did you do before writing code?**
Read the issue and the code, then commented with my proposed approach and asked
to be assigned, so the maintainers could agree on the scope.

**Q: What is contribution eligibility in Kaneo?**
A bot checks that the PR links an open issue labeled `ready-for-contribution`
(or that the author is vouched). Otherwise the PR is turned into a draft.

**Q: Why did you fix the slug bug in the same PR?**
It's the same root cause and a two-line change, so reviewers see the whole
fix at once. I mentioned it in the issue so maintainers could ask me to split it.

**Q: How did you use AI, and how do you make sure you understand the code?**
I used Claude Code for implementation and running commands. I reviewed the diff
line by line, verified tests failed before and passed after, and can explain
every decision. Kaneo's AI policy requires this, and PR text in your own words.

**Q: How do you handle review feedback you disagree with?**
Ask questions to understand the reasoning, explain mine with evidence (tests,
docs), and accept the maintainer's decision on their project.

---

## 9. Tricky follow-ups

**Q: Is the counter safe with multiple browser tabs?**
Yes. Each tab has its own JavaScript module instance and its own query cache, so
ids only need to be unique within a tab.

**Q: Two picker instances share the counter. Any problem?**
No, it's a benefit. They can never get the same id, even with the same token.

**Q: What if the same token is entered, removed, then entered again?**
Each change produces a new id, so it refetches. That's slightly less efficient
but always safe, and the old code behaved the same way.

**Q: What if the user signs out and someone else signs in with the same token text?**
`session.user.id` is also in the query key, and an existing test covers session
isolation.

**Q: Is the counter guessable? Does that matter?**
Yes, it's guessable, but it doesn't matter. It's not a secret and never leaves the
browser. Security comes from the key changing, not from unpredictability.

**Q: How would you stop this class of bug from coming back?**
A lint rule banning `crypto.randomUUID` in browser code, a test environment
that mimics a non-secure context, or an E2E test on a non-localhost HTTP origin.

**Q: Could you polyfill `crypto.randomUUID`?**
Yes, building a v4 UUID from `getRandomValues` (set the version and variant bits).
But that hides the issue globally and adds code; it's not needed when uniqueness
alone is the requirement.

**Q: Why not wrap the call in `try/catch`?**
It would need a fallback anyway, so it's the same as a fallback with extra
noise. Also, catching errors during render hides bugs.

**Q: Does the `apps/api` code need changing?**
No. It runs in Node.js, where `crypto.randomUUID` always exists. Secure
contexts are a browser concept.

**Q: What's the difference between availability and security in this fix?**
The bug was about availability (the API is missing). The fix had to keep a
security property (credential isolation). A fix that removed the crash but
reused cache keys would have introduced a data leak.

---

## 10. Behavioural (STAR)

**Q: Tell me about a bug you fixed.**
- **Situation:** self-hosted Kaneo users on plain HTTP couldn't use Gitea
  integration; the page crashed.
- **Task:** fix the crash without weakening the cache isolation between access tokens.
- **Action:** traced the crash to `crypto.randomUUID`, worked out that the id only
  needed to be unique, replaced it with a counter, found and fixed the same bug
  in slug generation with `getRandomValues`, and wrote regression tests first.
- **Result:** both crashes fixed, all existing security tests still pass, PR opened
  against a `priority:high` issue.

**Q: Tell me about a time you hit an unexpected obstacle.**
The pre-commit hook flagged 4,516 files I hadn't touched. I worked out that it
was Windows line endings, not my code, fixed my local git config, and avoided
reformatting the whole repo, which would have ruined the PR.

**Q: Tell me about something you learned recently.**
Secure contexts: some browser APIs vanish on HTTP. And the habit of asking
"what is this value *for*?" before replacing it.
