# Kaneo PR #2165: Fix sign-in page stuck on a loading skeleton after logout

| | |
|---|---|
| **Project** | [usekaneo/kaneo](https://github.com/usekaneo/kaneo) |
| **Issue** | [#2163](https://github.com/usekaneo/kaneo/issues/2163) |
| **Pull request** | [#2165](https://github.com/usekaneo/kaneo/pull/2165) |
| **Branch** | `man-singh-dev:fix/2163-signout-skeleton` |
| **Area** | Web app: authentication / sign-out (React, TanStack Query, TanStack Router) |

---

## 1. Problem statement

After clicking **Log out**, users were redirected to the sign-in page, but it showed a loading skeleton (three placeholder bars) forever. The sign-in form never appeared and no further network requests were made. Only a page reload fixed it.

**Steps to reproduce**

1. Sign in and stay on the dashboard for at least about a minute (or open the app in a new tab with an existing session).
2. Open the user menu and click **Log out**.
3. The app redirects to `/auth/sign-in` and shows the "signed out" toast, but only the skeleton renders.
4. Reloading the page shows the form.

**Expected:** the sign-in form appears immediately after logout.
**Actual:** the page stays on the skeleton until a reload.

### Root cause

The order of operations on sign-out was wrong:

1. `useSignOut` signed the user out and **navigated** to `/auth/sign-in`.
2. The sign-in page mounted and started fetching the `instance-status` query.
3. Only **after** that did `user-avatar.tsx` call `queryClient.clear()`.

`clear()` wiped the cache while the sign-in page's `instance-status` query was still in flight. The query was silently removed, the page never received a result, and it stayed in its loading state.

It only happened when `instance-status` was stale (after about a minute), which is why quick sign-in/sign-out cycles didn't reproduce it.

---

## 2. My original fix

- Moved the cache cleanup **into the `useSignOut` hook**, so it runs **before** navigating to sign-in (or before redirecting to the external identity provider for custom OAuth logout).
- Removed the late `queryClient.clear()` from `user-avatar.tsx`.
- Added tests in `use-sign-out.test.ts` covering:
  - the cache is cleared before navigating / redirecting,
  - the cache is **not** cleared if pending edits can't be saved,
  - the cache is **not** cleared if sign-out returns an error.

Having the cleanup in the hook also means every caller gets it. `onboarding-flow.tsx` also signs users out and previously did no cache cleanup at all.

---

## 3. Issue 1: CI failed, but not because of my code

The `quality` job failed at **Run typecheck**, and `ci-result` failed because it only aggregates the other jobs.

**Errors seen**

```
Cannot find module '@kaneo/editor' or its corresponding type declarations.
Property 'toggleBold' does not exist on type 'ChainedCommands'.
Binding element 'node' implicitly has an 'any' type.
Parameter 'updated' implicitly has an 'any' type.
...
```

None of these were in files my PR touched. They were all in `apps/mobile/src/features/task/rich-editor/`.

### Diagnosis

- A recent commit on `main` (`41a9dbcd`, "share the rich text schema between web and mobile") created a new workspace package, **`@kaneo/editor`**.
- That package exposes its types from `./dist/index.d.ts`. `dist` is build output and isn't committed, so it only exists after the package is built.
- The **web** app's typecheck task was configured to build its workspace dependencies first. The **mobile** app's typecheck task was not.
- In a fresh CI checkout, the mobile typecheck ran before anything built `@kaneo/editor`, so the module couldn't be resolved. Every other error followed from that: without the editor package, TypeScript also loses the Tiptap command types (`toggleBold`, etc.) and parameter types.

I confirmed this by running `pnpm typecheck` locally on my branch after deleting `packages/*/dist`. It reproduced the same errors, and `dist` was never built.

> **Lesson:** my first guess was that the CI workflow needed a "build packages" step. Reproducing it locally showed the real cause was the mobile app's task config. **Reproduce before you fix.**

### Fix

The maintainer had already fixed this on `main` in **`58c7bc7f` ("build(mobile): build deps before typecheck")**, which adds `dependsOn: build` to the mobile `check:types` and `test:run` tasks. It landed a few hours **after** my CI run, so my branch didn't have it.

**Solution:** merge the latest `upstream/main` into my branch.

> **Fork gotcha:** in my clone, `origin` is my fork (`man-singh-dev/kaneo`) and `upstream` is the real repo (`usekaneo/kaneo`). My fork's `main` was behind and had **neither** fix, so merging `origin/main` would have changed nothing. The merge has to come from **`upstream/main`**.

---

## 4. Issue 2: merge conflict with a rewrite on main

Merging `main` caused conflicts in my own files, because the maintainer had rewritten sign-out in **`6a9f6715` ("steady boot, navigation and loading")**.

**What main changed**

- Sign-out now **navigates to sign-in first, then ends the session**, with a rollback that returns the user to their page if sign-out fails. This avoids the app degrading through the 401 path.
- Cache cleanup stayed in `user-avatar.tsx`, but became **selective**: it removes every query **except** `config` and `instance-status` (which the new sign-in page reuses), and clears the mutation cache.

| | My PR | Main's rewrite |
|---|---|---|
| Order | sign out → clear cache → navigate | navigate → sign out → clean up |
| Cleanup | `queryClient.clear()` (everything) | remove all except `config` and `instance-status`, clear mutations |
| Where | inside `useSignOut` | in `user-avatar.tsx` only |

Keeping my full `clear()` would have deleted `instance-status`, the exact query from the original bug, and reintroduced a version of it.

### Resolution

I combined both approaches:

- **Kept main's navigate-first flow** and its rollback logic.
- **Moved main's selective cleanup into `useSignOut`** (my PR's idea), right after the session ends and before any IdP redirect:

  ```ts
  descriptionSaveQueue.clear();
  // Drop the signed-out user's data before leaving the app, so the
  // sign-in page (and the next session) never sees it. Keep the
  // instance config and status that the auth pages reuse.
  queryClient.removeQueries({
    predicate: ({ queryKey }) =>
      queryKey[0] !== "config" && queryKey[0] !== "instance-status",
  });
  queryClient.getMutationCache().clear();
  if (url) window.location.href = url.toString();
  else signOutState.isRedirecting = false;
  ```

- **Removed the cleanup from `user-avatar.tsx`**, since the hook now does it for every caller, including onboarding.

### Updating the tests

My test "clears the query cache before navigating to sign-in" could no longer be true, because main navigates first **on purpose**. I replaced it:

- **"drops user data once the local session has ended":** navigation happens before `signOut()`, cleanup hasn't run yet at that point, and afterwards queries and mutations are each cleaned up once, after the save queue is cleared.
- **"keeps the config and instance status the auth pages reuse":** the `removeQueries` predicate returns `false` for `["config"]` and `["instance-status"]`, and `true` for user data like `["tasks", "project-1"]`.
- **"clears the query cache before redirecting to the IdP":** kept unchanged.

The query-client mock was updated to provide `removeQueries` and `getMutationCache().clear()` instead of `clear()`.

---

## 5. Verification

Run from a clean state (`packages/*/dist` deleted):

| Check | Result |
|---|---|
| `pnpm install --frozen-lockfile` | ✅ |
| `pnpm typecheck` (all workspaces) | ✅ 0 errors |
| `pnpm lint` | ✅ no warnings or errors |
| Web tests (incl. `use-sign-out.test.ts`) | ✅ 1,148 passed (12/12 in `use-sign-out.test.ts`) |

Merge committed locally on `fix/2163-signout-skeleton`, then pushed to the fork to re-run CI on PR #2165.

**Follow-up:** the PR description had to be updated after the merge. It originally said the hook calls `queryClient.clear()` "before navigating to the sign-in page". After the merge, the hook navigates first and then removes everything except `config` and `instance-status`.

---

## 6. Issue 3: push rejected (non-fast-forward)

```
! [rejected]  fix/2163-signout-skeleton -> fix/2163-signout-skeleton (non-fast-forward)
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart.
```

### Diagnosis

```
git fetch origin
git log --oneline -4 HEAD
git log --oneline -4 origin/fix/2163-signout-skeleton
git log --oneline HEAD..origin/fix/2163-signout-skeleton
```

The last command showed exactly one commit on GitHub that wasn't local: `b7e76a20`.

The local branch and the PR branch held **the same code with different histories**:

- My fix was originally two commits (`af84fd83`, `e0c75266`).
- Earlier I had squashed them into one clean commit, `b7e76a20`, on a separate local branch `fix/2163-clean`, and pushed **that** as the PR branch.
- My local `fix/2163-signout-skeleton` still had the two old commits, and the upstream merge had been built on top of them.

Pushing would have discarded `b7e76a20` from GitHub, so Git refused.

### Fix: rebuild the merge on the published history (no force-push)

Instead of force-pushing over GitHub's version, I redid the merge on top of the clean commit and reused the already-tested conflict resolution (`29810319`):

```
git checkout fix/2163-clean
git merge upstream/main          # same two conflicts as before
git checkout 29810319 -- apps/web/src/hooks/mutations/use-sign-out.ts apps/web/src/components/user-avatar.tsx apps/web/src/hooks/mutations/use-sign-out.test.ts
git add -A
git commit -m "Merge upstream/main into sign-out cache fix"
git diff 29810319 HEAD --stat     # empty = identical to the verified merge
git push origin fix/2163-clean:fix/2163-signout-skeleton
```

- `git checkout <commit> -- <files>` copied the resolved files instead of resolving the conflicts by hand a second time.
- The empty `git diff` proved the new merge is byte-for-byte the code that passed typecheck, lint, and tests.
- The new merge builds on top of `b7e76a20`, so the push was a normal **fast-forward**. No `--force` was needed, and the PR keeps the single clean commit plus one merge commit.

---

## 7. Files changed (final diff vs `main`)

- `apps/web/src/hooks/mutations/use-sign-out.ts`: selective cache cleanup inside the hook
- `apps/web/src/components/user-avatar.tsx`: removed the duplicate cleanup
- `apps/web/src/hooks/mutations/use-sign-out.test.ts`: tests for cleanup order, preserved queries, and failure cases

---

## 8. What I learned

- **Order of side effects matters with query caches.** Clearing a cache after navigating can silently kill a query the next page is waiting on.
- **Clear selectively, not globally.** Some cached data (instance config/status) isn't user-specific and is needed by the logged-out screens.
- **CI failures aren't always yours.** Check whether the failing files are in your diff, then check whether `main` has moved on.
- **Reproduce locally before fixing.** My first theory about the CI failure was wrong. Running the exact command from a clean state found the real cause in minutes.
- **When main rewrites the code you touched, merge the intent, not the lines.** The best resolution kept the maintainer's new flow and my PR's goal (cleanup in the hook for every caller).
- **Keep PR branches up to date** with `upstream/main` before CI runs, especially in fast-moving repos.
- **In a fork, `origin/main` is not the real main.** Always sync from `upstream`.
- **Keep the PR description in sync with the code.** After a merge changes the approach, update the description so reviewers aren't misled.
- **Know which local branch matches the PR.** After squashing on a separate branch, either reset the original branch to it or keep working from the squashed one. Otherwise local and remote histories drift apart.
- **A rejected push is information, not an obstacle.** Check `git log HEAD..origin/<branch>` to see what you'd lose before reaching for `--force`.
- **Reuse a verified resolution.** `git checkout <commit> -- <files>` plus an empty `git diff` proves the redone merge is identical to the tested one.
