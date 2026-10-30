# Merge Conflict Resolution Cache

This repository stores a shared Git `rerere` cache under `rr-cache/`.

Git `rerere` means "reuse recorded resolution". When enabled, Git records how a
merge conflict was resolved once and can automatically apply the same resolution
when the same conflict appears again. This is useful for repeated kernel, Yocto,
or integration-branch merges that hit known conflicts.

## Repository Layout

```text
.
|-- README.md
`-- rr-cache/
    `-- <conflict-id>/
        |-- preimage      # conflicted file content seen by Git
        |-- postimage     # resolved file content recorded by Git
        `-- thisimage     # current conflict instance, when present
```

The `rr-cache/` directory is intended to be copied or linked into another Git
repository's `.git/rr-cache` directory.

## Quick Start

From the Git repository where you want automatic conflict resolution:

```bash
git config rerere.enabled true
git config rerere.autoupdate true
```

Then install this cache into that repository. Replace `/path/to/cache` with the
path to this repository.

```bash
mkdir -p .git/rr-cache
rsync -a /path/to/cache/rr-cache/ .git/rr-cache/
```

After that, run your merge, rebase, cherry-pick, or automerge workflow normally.
When Git recognizes a previously resolved conflict, it will reuse the recorded
resolution.

## Typical Workflow

1. Enable `rerere` in the target repository:

   ```bash
   git config rerere.enabled true
   git config rerere.autoupdate true
   ```

2. Copy this cache into the target repository:

   ```bash
   rsync -a /path/to/cache/rr-cache/ .git/rr-cache/
   ```

3. Run the merge or integration command.

4. Check whether Git reused a resolution:

   ```bash
   git status
   git rerere status
   git rerere diff
   ```

5. Review the resolved files before committing. `rerere` can save time, but the
   reused resolution still needs normal code review.

6. Commit only after the tree is correct.

## Adding New Resolutions Back To This Cache

When you resolve a new conflict manually in a target repository, Git records the
resolution in that repository's `.git/rr-cache`. To share it here:

```bash
rsync -a /path/to/target/repo/.git/rr-cache/ /path/to/cache/rr-cache/
```

Then commit the updated cache repository:

```bash
git status
git add rr-cache
git commit -m "Update rerere cache"
```

Before committing, inspect the changed files and make sure the recorded
resolution is correct and does not contain private or generated content that
should not be shared.

## Using A Symlink Instead Of Copying

If you want the target repository to use this cache directly, replace its
`.git/rr-cache` with a symlink:

```bash
rm -rf .git/rr-cache
ln -s /path/to/cache/rr-cache .git/rr-cache
```

Use this only when you are comfortable with the target repository writing new
records directly into this cache checkout.

## Useful Commands

```bash
git rerere status       # show paths with recorded conflict resolutions
git rerere diff         # show the current conflict resolution diff
git rerere remaining    # show conflicts still needing manual resolution
git rerere forget <path> # forget a bad resolution for one path
git rerere gc           # prune old rerere records using Git's expiry rules
```

## Safety Notes

- Always review automatically resolved files before committing.
- Do not assume every reused resolution is valid for a new branch or release.
- If a reused resolution is wrong, fix the file manually and run
  `git rerere forget <path>` before recording the corrected resolution.
- Keep this repository focused on `rr-cache/` records and documentation.
- Avoid committing unrelated build output, logs, or source trees.

## Troubleshooting

If conflicts are not resolved automatically:

- Confirm `rerere` is enabled with `git config rerere.enabled`.
- Confirm the target repository has records under `.git/rr-cache`.
- Run `git rerere status` after the conflict appears.
- Check whether the conflict is actually identical to a recorded conflict.
- Resolve it manually once; future identical conflicts can then be reused.
