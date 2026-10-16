# Testnet Documentation Link Audit

## Purpose

`testnet/` was built incrementally across many separate changes, which is
exactly how link rot starts: a document links to a heading that was later
renamed, or to a file that was planned and never written. This audit checks that
every relative cross-reference inside the tree resolves to a real file and, where
an anchor is given, to a real heading.

This document records the **method**, the **result of the first pass**, and the
**command to repeat it**, so the audit can be re-run after the rest of the
documentation lands rather than trusted from a single date.

## Status

**First pass complete for the tree as merged on 2026-09-26, plus the new `testnet/README.md` added in the same change.** Documents added by
changes that had not merged by then are not covered. Re-run the check after they
land (see [Re-running the audit](#re-running-the-audit)).

## Result

| Measure | Count |
|---|---:|
| Markdown files checked (first run) | 86 |
| Relative links checked (first run) | 667 |
| Broken on first run | 5 |
| Fixed in the same change | 3 |
| Still open | 2 |

### Findings

| # | Source | Link | Problem | State |
|---|---|---|---|---|
| 1 | `testnet/security/disclosure-process.md` | `#what-must-never-be-posted` | The heading is "What must never be posted publicly", so the anchor is `#what-must-never-be-posted-publicly` | Fixed |
| 2 | `testnet/security/disclosure-process.md` | `#what-would-count-as-making-this-heavier` | No such heading. The text it points to is a bold paragraph inside "This is not a mainnet disclosure policy" | Fixed: now `#this-is-not-a-mainnet-disclosure-policy` |
| 3 | `testnet/policies/reset-policy.md` | `#1-platform-level-reset-the-sdf-resets-testnet` | No such heading. The section is "Two independent kinds of reset" | Fixed: now `#two-independent-kinds-of-reset` |
| 4 | `testnet/docs/network-config.md` | `../../docs/testnet-deployment.md` | `docs/testnet-deployment.md` does not exist | **Open** |
| 5 | `testnet/docs/overview.md` | `../../docs/testnet-deployment.md` | Same missing file | **Open** |

### The two open findings

Both point at `docs/testnet-deployment.md`, a deployer-facing document that the
tree assumes exists and the repository does not contain. Fixing them is a
decision, not a typo: either that document should be written, or the two links
should be replaced with a pointer to `scripts/deploy-testnet.sh`. That choice
belongs to a maintainer, so this audit reports it and leaves both links as they
are.

## Method

The check reads every `*.md` file under `testnet/` and, for each inline link
`[text](target)`:

1. Skips absolute URLs (`http://`, `https://`, `mailto:`), which this audit does
   not fetch.
2. Resolves a relative path against the linking file's directory and requires the
   target to exist.
3. If the link has a `#fragment` and the target is a Markdown file, requires a
   heading whose GitHub-style slug matches it. The slug is the heading text
   lower-cased, with backticks and punctuation removed and spaces turned into
   hyphens. Repeated headings get a numeric suffix (`-1`, `-2`).

Links inside fenced code blocks and inline code spans are ignored, because they
are examples and not references.

### What this audit does not cover

| Not covered | Why |
|---|---|
| External URLs | They need the network and change independently of the repository |
| Reference-style links (`[text][ref]`) | None are used in the tree; add support if that changes |
| Links in JSON files | `postman-collection.json` and `last-verified.json` are data |
| Bare paths in backticks (for example `` `testnet/registry/contract-status.md` ``) | The tree's convention is that backticks **name** a path and links **open** a document, so a backticked path is not a cross-reference. A renamed file would still leave these stale, and this check will not see it |
| Documents added by changes that have not merged | They were not in the tree when this ran |
| Links from outside `testnet/` into it | Out of scope for this tree |

The backticked-path gap is the most important one. If a file under `testnet/` is
renamed, search the tree for its old path as plain text as well as re-running the
check.

## Re-running the audit

Save the script below as a file outside the tree (it is a one-off check, not part
of the documentation) and run it from the repository root:

```bash
python3 linkcheck.py . testnet
```

It prints JSON with the number of files, the number of links and a `broken`
list. An empty list means every checked link resolves.

```python
import os, re, sys, json
from urllib.parse import unquote
ROOT = sys.argv[1]
SCOPE = sys.argv[2:] or ['testnet']

def slug(h):
    h = re.sub(r'`', '', h.strip().lower())
    h = re.sub(r'<[^>]+>', '', h)
    h = re.sub(r'[^\w\- ]', '', h, flags=re.UNICODE)
    return h.replace(' ', '-')

anchor_cache = {}
def anchors(path):
    if path in anchor_cache: return anchor_cache[path]
    seen, out, fence = {}, set(), False
    try:
        for line in open(path, encoding='utf-8'):
            if line.lstrip().startswith('```'): fence = not fence; continue
            if fence: continue
            m = re.match(r'^(#{1,6})\s+(.*?)\s*#*\s*$', line)
            if m:
                s = slug(m.group(2)); n = seen.get(s, 0); seen[s] = n + 1
                out.add(s if n == 0 else f'{s}-{n}')
    except FileNotFoundError: pass
    anchor_cache[path] = out
    return out

LINK = re.compile(r'(?<!\!)\[([^\]]*)\]\(([^)\s]+)(?:\s+"[^"]*")?\)')
results = []; total = 0; files = 0
for scope in SCOPE:
    for dp, _, fns in os.walk(os.path.join(ROOT, scope)):
        for fn in sorted(fns):
            if not fn.endswith('.md'): continue
            fp = os.path.join(dp, fn); files += 1
            fence = False
            for i, line in enumerate(open(fp, encoding='utf-8'), 1):
                if line.lstrip().startswith('```'): fence = not fence; continue
                if fence: continue
                for m in LINK.finditer(re.sub(r'`[^`]*`', '', line)):
                    target = m.group(2)
                    if re.match(r'^(https?:|mailto:|#$)', target): 
                        continue
                    total += 1
                    path, _, frag = target.partition('#')
                    base = fp if path == '' else os.path.normpath(os.path.join(dp, unquote(path)))
                    rel = os.path.relpath(fp, ROOT)
                    if path != '' and not os.path.exists(base):
                        results.append((rel, i, target, 'missing file')); continue
                    if frag and base.endswith('.md') and os.path.isfile(base):
                        if slug(unquote(frag)) not in anchors(base) and unquote(frag).lower() not in anchors(base):
                            results.append((rel, i, target, 'missing anchor'))
print(json.dumps({'files': files, 'links': total, 'broken': results}, indent=1))
```

Run it after any change that renames, moves or deletes a file or a heading under
`testnet/`, and after a batch of new documents merges.

## Related Documentation

- [`README.md`](README.md): the entry point whose index this audit also checks
- [`policies/contribution-guide.md`](policies/contribution-guide.md): the link
  convention (links open documents, backticks name paths)
- [`docs/overview.md`](docs/overview.md): carries open finding 5
- [`docs/network-config.md`](docs/network-config.md): carries open finding 4

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | First link audit: 667 links checked, 5 broken, 3 fixed, 2 reported |
