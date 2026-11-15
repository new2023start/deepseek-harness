# Post-mortem 0005: A "stray directory" cleanup deleted an entire workspace root

English | [中文](0005-skill-home-consolidation-deletes-workspace-root.zh.md)

Status: resolved (data recovered)

## Executive summary

An agent session was consolidating duplicate `.claude` skill homes into a single canonical skills directory. Its PowerShell helper carried a non-ASCII path; PS 5.1 read the BOM-less UTF-8 script as ANSI, so the script created a mojibake sibling directory instead of writing into the intended workspace. The agent noticed, then wrote a cleanup that **discovered "the stray directory" by listing `E:/` and taking the first entry whose name starts with `00-`**. The only such entry was the real workspace `E:\00-学习`; the cleanup adopted it as the stray tree, renamed entries through it, and the workspace was recursively deleted six seconds later — 55,064 files, 13.8 GB. The lesson: a destructive target may never be *derived* from a pattern; it must be the exact canonical path that was created, or the operation must abort.

## Summary

The consolidation goal was legitimate: six skill directories existed twice, once under `C:\Users\Administrator\.claude\skills` and once under `C:\Users\Administrator\.openclaw\workspace\skills`. The plan was to move each `.claude` copy into a dated trash directory inside the user's workspace and leave a junction pointing at the canonical copy.

The helper script embedded a Chinese path (`E:\00-学习\_trash\...`). Because the script was written as UTF-8 without a BOM and executed with `powershell -File`, Windows PowerShell decoded it with the ANSI code page. The script therefore created a *different* directory whose name is the mojibake rendering of the same bytes, while every path it printed still looked correct in the terminal.

The agent detected the mismatch, tried to relocate the six directories back into the real trash path, and hit `ENOENT` on every source. It then rewrote the relocation to discover the mojibake root dynamically:

```js
const roots = fs.readdirSync('E:/', { withFileTypes: true })
  .filter((e) => e.isDirectory() && e.name.startsWith('00-'))
  .map((e) => e.name)
```

The mojibake directory had already been removed or never matched the filter, so `roots` contained only the real workspace. The script logged `stray root = 00-学习`, attempted `rename()` of every entry from the real path into the mojibake path, and the deletion that followed removed the workspace itself.

## Impact

- 55,064 files (13.8 GB) deleted from the workspace: notes, project sources, documents, agent memory.
- The deletion bypassed the Recycle Bin and the volume had no shadow copies, so normal undo was unavailable.
- Items that were open at deletion time were renamed into `$Extend\$Deleted` before removal; those records were reclaimed and could not be recovered.
`winfr` (Microsoft Windows File Recovery) in regular mode restored 55,065 directory entries with their original names, paths, and sizes. That is not the same as restoring data: this volume is a TRIM-enabled NVMe SSD (`fsutil behavior query DisableDeleteNotify` returns 0), so the controller had already zeroed the freed LBAs. Sampling the recovered tree afterwards, 302 of 400 files began with zero fill, and 0 of 30 PDF/DOCX/xlsx files carried a valid signature. Only MFT-resident files survived: 12,004 of the 12,836 files at or below roughly 700 bytes still held intact content (3.0 MB total). The remaining 42,229 entries are name-and-size stubs whose blocks are gone.

## Timeline

- The agent wrote a consolidation helper and started it from `run_code`. The script's only non-ASCII path was the workspace trash root.
- The helper created a mojibake sibling of the workspace and moved the six skill directories there, then created junctions at the sources.
- The agent noticed that the intended trash path was still empty, probed the `E:` root, and found the mojibake tree.
- A relocation script moved the six directories back by exact path. The first attempt failed with `ENOENT`; a rewrite replaced the exact path with a `readdirSync('E:/')` prefix filter.
- That rewrite reported `stray root = 00-学习` and issued renames from the real workspace into the mojibake path.
- USN journal records show the workspace tree deleted between 11:32:29 and 11:32:35, with the top-level directory removed last.
- Recovery began at 11:53. `winfr` regular mode restored 55,065 files by original path; free-space carving found nothing, confirming the remaining clusters were already reclaimed.

## Root cause

**Non-ASCII paths do not survive a BOM-less script round-trip.** `powershell -File` decodes a BOM-less script with the ANSI code page. Embedding `E:\00-学习` in such a script silently changes the path; every message the script prints is decoded back correctly, so the mismatch is invisible in the transcript. Any tool that both *writes* and *deletes* by such a path must verify that the path it created is the path it later addresses.

**A destructive target was derived from a pattern.** The cleanup needed "the directory I accidentally created", which is knowable exactly: it is the path the helper logged. Replacing that with "the first `E:/` entry whose name starts with `00-`" made the workspace a candidate. When the real stray entry disappeared, the pattern silently selected the real workspace. The operation had no invariant tying the target to the object that had been created, and no check that the target was not the workspace root.

**The two failures composed.** The mojibake created a tree that should not exist; the prefix discovery then chose the one directory that must never be deleted. Either guard alone would have prevented the loss.

## Guardrails added

- Never construct a destructive path from a pattern. Pass the exact created path into the cleanup, and abort if it does not exist.
- Before any recursive delete, assert the target is not a workspace root, not a parent of the current working directory, and not reached through a junction.
- Treat "the path I just created" and "the path I am about to delete" as a single value that travels together; do not let the model recompute it later.
- When a non-ASCII path must cross a shell boundary, write the script with a UTF-8 BOM (or pass the path as an argument), and verify existence immediately after creation.
- For agent-driven consolidation, prefer copy → verify → delete-last, so a failed verification leaves the source intact.

## Lessons

- A path that renders correctly in a transcript is not evidence that the bytes are correct.
- "Find it by prefix and delete it" is an unsafe idiom even when the predicate looks unambiguous; breadth of the predicate is not the problem, derivation itself is.
- Cleanup scripts written in response to an error are the highest-risk code an agent produces: they run unattended, target user data, and are usually untested.
- Recovery cost dwarfs prevention cost: this incident consumed roughly an hour of forensic work plus two full-volume passes, and still recovered no document larger than a kilobyte.
- On an SSD, deletion is destructive immediately. MFT recovery restores names and sizes, which makes the result look successful until the contents are checked; "the files are recent, so they should be recoverable" is false under TRIM.
