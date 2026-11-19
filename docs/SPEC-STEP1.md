# EditAnt: SPEC STEP 1 (2026-11-19, docs only; no code until CireSnave has read this)

Builds on `SPEC-SEED-2026-11-19.md` (markers **[said]** / **[PM]** there). Here: **[verified]** = run or read by the lane this session; **[seen]** = seen in a search result only, not opened.

## 1. Survey

### 1.1 Markdown parsers (crates.io metadata fetched 2026-11-19 from the crates.io API)
| crate | version | licence | MSRV | note |
|---|---|---|---|---|
| `pulldown-cmark` | 0.13.6 | MIT | 1.71.1 | pull parser, `into_offset_iter()` gives byte ranges |
| `comrak` | 1.0.0 | BSD-2-Clause | 1.89 | AST with source positions (line/col), GFM; heavier |
| `markdown` (markdown-rs) | 1.0.0 | MIT | 1.56 | mdast with positions |
| `markdown-it` | 0.6.1 | MIT | none | last release 2024-07 |

**[verified]** a probe (scratch crate, not committed) ran `pulldown-cmark` 0.13.6 with `ENABLE_YAML_STYLE_METADATA_BLOCKS` on
`---\ntitle: x\n---\npre\n# A\n```\n# not\n```\nSetext\n======\nbody\r\n## B ##\n`: it returned a metadata block `0..16`,
`# A` at `21..25`, the setext header at `39..53` and `## B ##` at `59..67`; the `# not` inside the fence produced no heading.
Offsets are byte ranges that include the header's trailing newline, and a mixed LF/CRLF input parsed. So the seed's section-boundary
needs (ATX, setext, fences, front matter) are met by `pulldown-cmark` alone. Not verified: HTML blocks, `+++` TOML front matter,
headings in block quotes/lists (CommonMark: a `#` inside a list item or quote is a heading *inside that container*; decision D3).

### 1.2 Existing section editors (all **[seen]** via web search 2026-11-19; none opened, none run; boundary rules unchecked)
- **md-mcp** (afriemann): MCP, sections addressed by dot-separated heading path, per-section read/edit.
- **nodiom-mcp**: MCP, selectors like `# Heading > ## Subheading`, a tree/outline tool.
- **basic-memory** `edit_note`: `replace_section` op among append/prepend/find-replace.
- **obsidian-mcp** (adrienthebo): replace, insert-before/after, append-to-section.
- **quantalogic SafeMarkdownEditor**: transactions + validation; delete by section ID or heading.
- **texio-cli** (Rust): `section` and `replace --section --dry-run --write`; refuses ambiguous headings; self-described early preview. Closest to our MVP.
  **[read, not run]** 2026-11-19, texio-cli 0.1.3 (MIT, `Allra-Fintech/texio` @ `4ea7098`, cloned to scratch; the build was denied by the session's permission classifier, so NOTHING was executed): `docs/markdown-support.md`, `docs/cli-contract.md`, `Cargo.toml`, `src/main.rs` (826 lines) read. Against our four differentiators:
  (a) **token-bound preview + stale refusal: not found.** `replace --dry-run` shows a diff; grep of `src/` and `docs/cli-contract.md` for hash/token/stale/checksum/sha/blake finds no hit (control: the same grep finds "ambiguous" and "dry-run", which are present). Not run, so "absent" means "not in the source read".
  (b) **child-excluding default extent: no, the opposite.** README: "The replacement covers the complete selected body, including nested subsections"; sections run to the next heading of the same or higher level.
  (c) **byte-for-byte preservation:** claimed (LF/CRLF, final newline, front matter untouched, atomic write, permissions kept; unit tests for CRLF and setext). No `proptest`/`quickcheck` in `Cargo.toml`, `src/main.rs`, `tests/` (control: `Cargo.toml` listing shows its real dev-dependencies). Claims untested by us.
  (d) **ambiguity refusal: yes** (exit code 5). Lookup is ASCII case-insensitive; "path-like selectors for duplicate headings are not implemented yet" (its own limitations list).
  Also: it recognises YAML front matter and skips HTML blocks and fences; it uses `pulldown-cmark` 0.13. **Consequence:** (d) is not a differentiator, (b) is a deliberate difference (our D-extent default), (a) and header-path addressing are the open ones. The README must not claim novelty for the idea or for ambiguity refusal.
- Rust read-only/other: `mdq` (jq-like queries), `mdlens` (section read for agents), `treemd` (viewer, interactive editing), `kugiri` (marker-based blocks).

**Reading:** the idea is not new. Differentiators we can still earn: (a) preview bound to a hash token with stale-edit refusal, (b) default extent excludes child sections, (c) byte-for-byte preservation with property tests, (d) refusal of ambiguity. **Whether any competitor already does (a)-(d) is unverified**; before the README claims novelty, open texio-cli and md-mcp and run them (task S1).

## 2. Scope (MVP, as the seed section 2, refined)
Library crate `editant` + CLI `editant`; MCP wrapper later.

Operations (all take a file path and a **header path**):
1. `outline(file)` -> list of `{path, level, index, line, kind: atx|setext|preamble}`.
2. `read_section(file, path[, subtree])`.
3. `preview(file, path, replacement|none[, subtree])` -> `{original, replacement, diff, covers_children: n, token}`; writes nothing.
4. `apply(token)` -> writes atomically (temp file + rename), fails with `Stale` if the file's current bytes hash differently from the preview's.
5. Delete = `preview(..., none)`.

**Addressing:** `Parent > Child`, comparison on the header's rendered text with surrounding whitespace and closing `#`s trimmed, case-sensitive; optional `#n` occurrence index (1-based). Zero or several matches without an index: `Err(Ambiguous{candidates})` listing each candidate's line. Never guess.

**Extent:** own body only, header line up to the next header of any level; `subtree` flag extends to the next header of the same or shallower level and the preview reports the child count. The header line itself is kept by default (replacement is the body); decision D2 covers replacing the header too.

**Token:** `blake3`/`sha2` of {canonical file path, section byte range, hash of original section, hash of replacement}. `apply` re-reads the file and recomputes; any mismatch refuses.

**Preservation:** untouched bytes are copied, never re-rendered; the replacement is spliced at the byte range. Line ending of the replacement is normalised to the file's dominant ending (D4); final-newline and BOM state preserved. Property tests: identity replacement returns identical bytes; bytes outside the range unchanged for any section of any generated document; parser-based oracle (outline after edit differs only at the target).

## 3. Non-goals (this step)
TOML/JSON, MCP server, GUI, merging, history/undo, human-approval UI, insert-before/after and header rename (next, listed in the seed), any model call.

## 4. Crate shape and constraints
`#![forbid(unsafe_code)]`; dependencies permissive only, latest versions at all times (CireSnave's dependency rule); `pulldown-cmark` kept with its MIT notice in a `THIRD-PARTY` file; MSRV raised as needed. Single crate with `lib` + `bin` first; split only when the MCP layer exists. Dual MIT/Apache-2.0 as the seed's licence files.

## 5. Plan after approval
S1 survey close-out (run texio-cli and md-mcp, record results) -> S2 parser/section model + outline + tests -> S3 preview/apply/token + property tests with sabotage checks -> S4 CLI -> S5 README claims earned by tests. One PR per day of work per the portfolio rule.

## 6. Unverified, stated plainly
Competitor behaviour (1.2; texio-cli read but not run, md-mcp and the others not opened); parser behaviour for HTML blocks, TOML front matter and container headings; Windows atomic-rename behaviour under antivirus locks; whether `pulldown-cmark` offsets for setext headers with trailing spaces or lazy continuation lines are exact (probe covered only the simple case).

## 7. Decisions for CireSnave
- **[ASK] D1 Name matching:** case-sensitive exact (recommended) vs case-insensitive. Blocks: S2.
- **[ASK] D2 Header line:** a replacement keeps the header and replaces the body (recommended; deletion of a section removes header and body), vs the replacement text includes the header so renaming is possible in one step. Blocks: S3.
- **[ASK] D3 Headers inside lists/block quotes:** treat as non-addressable (recommended; they are not document structure) vs addressable. Blocks: S2.
- **[ASK] D4 Line endings in replacement:** normalise to the file's dominant ending (recommended) vs insert exactly as given. Blocks: S3.
- **[ASK] D5 Approval:** MVP keeps the LLM's own approve step as the token-bound `apply`; add an optional human-confirm hook later (recommended), or build it now. Blocks: S4.
- **[ASK] D6 Crate/binary name** `editant` on crates.io (not checked for availability by this lane). Blocks: first publish only.

— lane: editant
