---
layout: post
title: "CHANGELOG.md Doesn't Merge Itself"
date: 2026-10-12
categories: [devops, automation, ci-cd, git]
---

## How We Stopped Fighting Merge Conflicts on CHANGELOG.md

There's a specific type of merge conflict that every team using semantic versioning experiences. 

Or... at least I hope it's not just us.

Two admins working on separate branches both need to update CHANGELOG.md. One adds version 2.3.0 for their new feature. The other adds version 2.3.0 for a different feature. Both push to their branches, both create merge requests, and the second one to merge gets a conflict.

The conflict itself is trivial to resolve, but the problem runs deeper. Now someone has to decide which change becomes 2.3.0 and which becomes 2.4.0. If you picked version numbers before you knew the merge order, you picked wrong. If you wait to pick version numbers until you know the merge order, you're coordinating version allocation across your team for every change.

After months of "can you rebase and update your version number?" comments on merge requests, we stopped fighting the problem and built a system that made it impossible. The result is a changelog fragment system that eliminates merge conflicts and calculates version numbers automatically based on what actually merged.

## The Coordination Tax

When we moved our Jamf Pro infrastructure to Terraform and adopted semantic versioning, we inherited a workflow that looked sensible on paper:

1. Create a feature branch
2. Make your changes
3. Update CHANGELOG.md with the next version number
4. Commit and push
5. Create a merge request
6. Merge when approved

This workflow has a hidden assumption: that merge requests merge in the order they're created. The problem is, that's not always how it works. Merge order often depends on review speed, approval requirements, and which changes are ready first.

Consider what happens when two admins start work on the same day. Admin A creates a branch for a new policy deployment. Their change adds features, so they update CHANGELOG.md with version 2.3.0. Admin B creates a branch to fix a bug in a smart group. Their change is a bug fix, so they update CHANGELOG.md with version 2.2.9 (a patch bump from the current 2.2.8).

Admin B's fix gets reviewed faster because it's small and obviously correct. It merges first. The repository is now at version 2.2.9.

Admin A's MR now has a conflict in CHANGELOG.md. The file shows version 2.3.0, but it's branched from 2.2.8, and the current version is 2.2.9. Even worse, the version number is now wrong. Admin A's feature should be 2.3.0, but it needs to be based on 2.2.9, not 2.2.8.

Admin A rebases, fixes the conflict, updates the version number again, and force-pushes. The MR merges. The cycle repeats with the next pair of branches.

Every merge conflict is trivial. Every version number correction is trivial. But the coordination overhead is constant. You can't work independently because version numbers create dependencies between unrelated changes.

## What We Tried First

We looked at the conventional solutions. Tool-release versioning (where CI picks the next version at release time) just defers the problem. Admins still need to document their changes somewhere, and if it's not in CHANGELOG.md, it's in commit messages or pull request descriptions. You've just moved the work from development time to release time, where it happens under pressure and with less context.

Conventional commits would work if our commit messages were detailed and well-formatted. They aren't. Terraform changes often have commit messages like "Update policy" or "Fix smart group criteria" because the Terraform plan output shows the actual change. More importantly, you can't enforce conventional commit formats in CI the way you can enforce other checks. Tools like commitlint run as local git hooks that can be bypassed with `--no-verify`. By the time the commit reaches CI, the message is already written and part of git history.

We could skip the changelog entirely and tie all releases to tickets in our ITSM — rely on ticket history for release documentation instead. However, our ITSM doesn't have native GitLab integration, so we'd need to build that ourselves. The fragment system was simpler to implement than an integration layer. There's also the risk of losing your changelog if you ever switch ITSM platforms. We're keeping this option in mind for the future, but it wasn't the right fit now.

[TownCrier](https://towncrier.readthedocs.io/en/stable/index.html) is an existing Python tool that solves this exact problem with changelog fragments. It's mature, well-tested, and handles fragment compilation with more features than we needed. But it felt like overkill. We wanted something simple that integrated directly into our existing CI pipeline without adding dependencies or learning a new tool's configuration system.

Manual conflict resolution was what we were doing. It worked in the sense that the conflicts were resolvable, but the coordination overhead scaled with team size. The answer to "who should resolve this conflict?" is always "whoever merged second," which means the cost is unpredictable and falls on whoever happened to be slower.

We needed something that eliminated coordination entirely. If two admins work on separate changes, they shouldn't need to communicate about version numbers or changelog order. The system should figure it out.

## Changelog Fragments

The solution we landed on is simple. If merge conflicts happen because multiple branches edit the same file, stop editing the same file. Instead of updating CHANGELOG.md directly, admins create small "fragment" files that describe their changes. The CI pipeline compiles those fragments into CHANGELOG.md after the merge happens.

A fragment file is a markdown file with YAML frontmatter. Here's an example:

```markdown
---
type: changed
scope: Production
title: Jamf Connect Detection
incident: INC-12345
---
With the advent of Self Service+, Jamf Connect is no longer available as a standalone app. Deploying EA to all servers and adjusting smart groups for proper detection.
```

The frontmatter holds metadata about the change: what type it is (`added`, `changed`, `removed`, or `fixed`), which server or area it affects, a short title, and optionally a reference to a ticket. The body contains the description of the change.

The filename doesn't matter much as long as it's descriptive, lowercase, and uses only letters, numbers, and hyphens. `jamf-connect-detection.md` works. So does `fix-bug-123.md`. The name helps with git history but doesn't affect the compiled output.

When you open a merge request, the CI pipeline validates that the fragment exists and is properly formatted. If validation passes, the MR can proceed to review. If validation fails, the CI comments on the MR with specific errors: missing frontmatter, invalid type, empty body - whatever's wrong.

After the MR merges to main, a separate CI job runs. It reads all the fragment files in `changelog.d/`, compiles them into a new section in CHANGELOG.md, calculates the next version number based on the fragment types, commits the updated CHANGELOG.md with a message like `[release] v2.3.0`, and creates a git tag.

Version calculation follows semantic versioning rules, but the logic runs after merging instead of before. If any fragment has `major: true`, bump the major version. If any fragment has `type: added`, bump the minor version. Otherwise, bump the patch version. Because calculation happens after merging, the version number reflects reality. If three MRs merge in quick succession (two patches and a minor), the version goes 2.2.8 → 2.2.9 → 2.2.10 → 2.3.0. The order is determined by merge order, not by what someone guessed days earlier.

The compiled CHANGELOG.md entry looks like this:

```markdown
## [2.3.0]

**Release** [v2.3.0](https://gitlab.com/org/repo/-/tags/v2.3.0)

### Changed
- **Production** - **Jamf Connect Detection** - With the advent of Self Service+, Jamf Connect is no longer available as a standalone app. Deploying EA to all servers and adjusting smart groups for proper detection. (INC-12345)
```

Previous versions get wrapped in `<details>` tags so the changelog file doesn't grow unbounded. The most recent release is always visible. Older releases are collapsed but still accessible.

## The Implementation

The whole system is under 500 lines of Python: a validator that checks fragment files on merge requests and a builder that compiles them into CHANGELOG.md on the main branch. The logic is simple enough that we wrote it in an afternoon.

The validator reads fragment files, parses the YAML frontmatter, checks that required fields exist, and validates the format. If anything's wrong, it comments on the MR with specific errors. If everything looks good, it passes and the MR can proceed.

```yaml
check-changelog:
  stage: test
  script:
    - python -m changelog_tools.changelog.validator
  only:
    - merge_requests
```

The builder runs after merges to main. It collects all the fragment files, groups them by type, generates a new changelog section with the calculated version number, wraps the previous version in `<details>` tags, and commits the result.

```yaml
build-changelog:
  stage: release
  script:
    - python -m changelog_tools.changelog.builder
    - git add CHANGELOG.md
    - git commit -m "[release] v${VERSION}"
    - git tag "v${VERSION}"
    - git push origin main --tags
  only:
    - main
```

The version calculation logic fits in a method:

```python
def calculate_next_version(current: str, fragments: List[Dict]) -> str:
    major, minor, patch = map(int, current.split("."))

    has_major = any(f.get("major") for f in fragments)
    has_added = any(f["type"] == "added" for f in fragments)

    if has_major:
        return f"{major + 1}.0.0"
    elif has_added:
        return f"{major}.{minor + 1}.0"
    else:
        return f"{major}.{minor}.{patch + 1}"
```

## How It Works in Practice

Fragment files never conflict because each branch creates a new file with a unique name. Two admins working on separate branches create `feature-a.md` and `bugfix-b.md`. Both merge without touching the same file. No coordination required.

Version numbers are calculated after merging based on what actually made it into the branch, so they're always correct. No more guessing what version number to use when you create a branch. No more rebasing to fix version numbers when merge order changes. The system figures it out.

Creating a changelog fragment is faster than updating CHANGELOG.md manually. You don't need to figure out the next version number. You don't need to worry about merge conflicts. You don't need to coordinate with other branches. Create a file, write a description, commit, push.

The CI validation catches mistakes early. Admins get immediate feedback in the MR if the fragment is malformed or missing required fields, rather than discovering problems at merge time.

Fragment descriptions are better than commit messages. Commit messages are written for admins reading git history. Changelog descriptions are written for users or operators who need to know what changed. Separating those audiences produces better documentation for both.

## What We Learned

We didn't iterate much on this system. The initial design worked, and we haven't needed to change it. Fragment files eliminated the coordination overhead. Version calculation became automatic. The validation gave us confidence that fragments were well-formed before they merged.

The main thing we learned is that the problem was coordination, not conflict resolution. The merge conflicts themselves were trivial. The real cost was coordinating version numbers across branches and rebasing when merge order didn't match creation order. Fragment files eliminate coordination entirely because each branch works in isolation.

On a small team, changelog conflicts are occasional annoyances. As the team grows and merge request volume increases, they become constant friction. Fragment files scale linearly because they're independent by design. Two admins working on separate branches never conflict. They each create their own fragment file with their own unique filename.

The system is transparent. Every MR shows whether the fragment is valid. The compiled changelog shows exactly what got released and when. The version number calculation is deterministic and based on what actually merged rather than what someone predicted would merge.

Collapsed previous versions keep the changelog readable. Wrapping old versions in `<details>` tags means CHANGELOG.md doesn't become a multi-thousand-line file that nobody reads. The latest release is prominent. Historical releases are available but don't clutter the view.

## Worth Considering

If you're managing infrastructure as code and experiencing changelog conflicts, this pattern is worth considering. The implementation is simple - the payoff shows up immediately on the first conflicted merge and the coordination overhead disappears.

The specific implementation we use is Python-based and integrated with GitLab CI, but the pattern is language-agnostic and CI-agnostic. The core idea is: admins create small files instead of editing a central file, CI validates those files on merge requests, CI compiles those files into the central changelog after merging, and version numbers are calculated from the compiled result.

You could implement this with shell scripts, GitHub Actions, Jenkins, or any CI system that can run scripts and commit changes. Or you could use TownCrier if you want a mature tool with more features. The pattern matters more than the implementation.

If you're coordinating version numbers across branches, you're spending time on a problem that doesn't need to exist. Fragment files make the problem impossible instead of just easier to resolve.

---

*Built with Python and GitLab CI in under 500 lines of code.*
