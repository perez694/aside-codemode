<p align="center"><img src="assets/logo.png" alt="aside-codemode" width="112"></p>
<h3 align="center">code mode over Aside's native REPL</h3>
<p align="center"><b>Search, read and filter in one call, so only the answer reaches the model</b><br>
Aside ships ripgrep and no documented way to reach it. This puts it on the surface,<br>
along with parallel browsing, through one code-mode tool.</p>

<p align="center">
  <a href="https://www.npmjs.com/package/aside-codemode"><img src="https://img.shields.io/npm/v/aside-codemode?color=cb3837&label=npm&logo=npm" alt="npm version"></a>
  <a href="https://github.com/lidge-jun/aside-codemode/blob/main/LICENSE"><img src="https://img.shields.io/npm/l/aside-codemode?color=blue" alt="license"></a>
  <img src="https://img.shields.io/node/v/aside-codemode?logo=node.js&label=node" alt="node version">
  <a href="https://github.com/lidge-jun/aside-codemode/actions/workflows/ci.yml"><img src="https://github.com/lidge-jun/aside-codemode/actions/workflows/ci.yml/badge.svg?branch=main" alt="ci"></a>
</p>

```bash
npm install -g aside-codemode
codemode --install-mcp
codemode --doctor
```

<p align="center"><a href="README.md">English</a> · <a href="README.ko.md">한국어</a></p>

### Browsing: 1.87 MB in, 4.4 KB out

The table below comes from three runs on 2026-09-15. The [measurement
note](evidence/browse-compression-260915.md) records the method, per-page bytes, timings and
the pages excluded from the set.

Five pages, and one thing to know about each — its title and its first link.

|  | reading the pages | one `browse.exec` call |
| --- | --- | --- |
| what a fetch-based tool puts in the model | 1,865,043 bytes of HTML | 4,430 bytes of typed rows |
| what an agent reading natively puts there | 71,983 chars of accessibility tree, three of five cut off at the cap | the same 4,430 |
| round trips | five | one |
| wall clock, five pages | — | 2.5s |

```js
const res = await browse.exec({
  urls,
  extract: { title: 'title', firstLink: { selector: 'a', attr: 'href' } },
});
return res.items;   // five typed rows, 4,430 bytes, status completed
```

**421x against the raw pages, 16x against what Aside itself would have shown the model.** The
[2026-09-15 measurement](evidence/browse-compression-260915.md) includes the per-page bytes and
the two pages that were dropped because they answered with a captcha. The
compression is of the **answer**: ask for the whole page and you get the whole page. Three of
the five accessibility trees hit the 20,000-character cap, so on those pages the native path was
not holding a complete answer either.

### Files: 55s becomes 1s

Find the fifty files in a project that mention `TODO`, and hand back the paths.

|  | natively | one `codemode --code` call |
| --- | --- | --- |
| cards in the Aside UI | 50 | 1 |
| round trips | 50 | 1 |
| what reaches the model | every byte of all fifty files | the five paths you asked for |

```js
const hits = await search.content({ path: ".", query: "TODO", max: 50 });
return [...new Set(hits.rows.map((r) => r.file))].slice(0, 5);
```

On a real development folder, `find`+`grep` took **55s** and one `codemode --code` search took
**1s**, about **51x**. That pair is an [operator report](evidence/dev-folder-51x.md), not an
independently reproduced measurement; the note carries no measurement date and says that the
folder, unrounded clocks and exact options were not recorded.

The next two comparisons come from the controlled 2026-09-18 MCP measurement against codemode
0.8.1, N=7 after a discarded warm-up, with result counts checked on both sides. Method, full
tables and the cells where this loses: **[BENCHMARKS.md](BENCHMARKS.md)**.

That run reproduced the shape of the operator report and found something sharper. Asked to
count `function` across a 127,000-file tree, `grep -r -I` did not finish: it passed **128
seconds having emitted 3.23 GB across 3.7 million lines** and was aborted. `search.count`
answered in **2,966 ms** and returned **778 bytes** — 502,963 matching lines in 57,403 files.

That is not a speedup. It is a question an agent holding POSIX tools cannot ask, because the
output buries the conversation before the answer arrives. Listing every `*.md` in the same tree
is the plainer version: `find` 5,467 ms against **2,063 ms**, same 39,831 paths.

Older paired Aside-turn timings (model + daemon overhead) were 1.05–1.81x for single searches. Those do not cancel the folder wall-clock. [See the older table](#performance-evidence).

### The engine was already there

A 2026-09-18 [attachment check](evidence/aside-mcp-attach-260918.md) found Aside's bundled
ripgrep reporting version 15.2.0 with PCRE2 at `runtime/native/bin/rg`, and **no documented tool
reaches it**. That path is the skill runtime's utility
bin: `rg` sits there beside `pdftotext`, `pdftoppm`, `python3` and `node`, vendored from
Homebrew (`runtime/manifest.txt`). The documented agent tools are `read_file`, `write_file`,
`edit_file`, `bash` and `repl`, and none of them is a content search. An agent that types `rg`
in bash is using an undocumented implementation detail, not a provided tool.

So an agent's real options were POSIX `grep` and `find` through bash, which is why the numbers
above are measured against those and not against ripgrep. Ripgrep is the ceiling, not the
baseline. This package puts that engine on the surface as 46 guest actions behind one tool.

Against ripgrep called directly the two are level, because `search.*` shells out to that same
binary. This is not a faster search engine. It is the engine Aside already shipped, reachable,
with the result filtered before it reaches the model.

### Code mode over Aside's REPL

`aside-codemode` compiles each browser batch into JavaScript and invokes `aside repl`.
[`session.js`](src/host/browse/session.js) starts that command, and
[`script.js`](src/host/browse/script.js) builds the code passed to it.

This package exposes one tool for batched pages and filtered results over that REPL path.

**aside-codemode** gives Aside a single place to search, filter, read and summarize local files —
and, once browsing is turned on, to run twenty pages through one session and come back with rows
instead of screenshots. Keep intermediate data out of the model context; return the answer and the
evidence needed to judge it.

Native MCP is the first-class route to code mode: after its tool inventory has been cached, Aside
attaches `mcp__aside-codemode__execute_code` directly. The supported CLI route follows for hosts
that do not attach MCP servers and for people who prefer a visible `bash` card. File cards in the
Aside UI still come from native `read_file` / `write_file` / `edit_file`. Guest JavaScript uses
those same shapes.

## Requirements

- Node.js >= 18
- ripgrep for the CLI route. The MCP route resolves Aside's bundled native ripgrep automatically on macOS and Windows; `CODEMODE_RG` / `rgPath` remains an explicit override
- macOS and Windows

## Install

Install the package once, then set up Route 1, native MCP. Use Route 2 when the host does not
attach MCP servers or when you prefer account guidance and a visible `bash` card.

```sh
npm install -g aside-codemode
codemode --install-mcp
codemode --doctor
```

From a clone instead, which is what you want if you are going to change it:

```sh
git clone https://github.com/lidge-jun/aside-codemode.git
cd aside-codemode
npm install -g .        # or: npm link
codemode --doctor
```

### Route 1: native MCP

Aside ships an MCP client. The installed CLI registers `aside-codemode`, asks the live Aside
daemon to reset its tool-inventory migration, runs one ordinary discovery session, and verifies
the inventory Aside wrote:

```sh
codemode --install-mcp --account u1
```

`--account` is optional; use the `u<n>` name from `~/.aside/u/`. Add `--json` for a machine-readable
report. On the measured macOS run, with Aside CLI 1.26.906.1630, the live settings write exited 0
and read back migration version 0 with an empty inventory map. The discovery session exited 0,
and `settings.json` then contained `inventories["aside-codemode"].tools = ["execute_code"]`.
The [first measured run](evidence/aside-mcp-activation-260918.md) took 17 seconds and a later
run against an already-registered server took 3, neither opening Settings nor restarting the
daemon. The same command was measured on Windows from a global install, where it returns
`activated: true` with `execute_code` cached.

The MCP server resolves Aside's bundled native ripgrep automatically on macOS and Windows.
Set an absolute `rgPath` in codemode config or `CODEMODE_RG` only when you want to override it.
The command does not fabricate an inventory: the Aside daemon discovers and caches
`execute_code`, which is attached as `mcp__aside-codemode__execute_code` in a new session.

The daemon's `set()` replaces the whole `mcp` object, so this operation drops every other
server's cached inventory. Discovery then visits those servers again; Aside disables one it
cannot reach and does not retry it automatically. `codemode --install-mcp` therefore refuses
when another enabled MCP server or another cached inventory exists. It names what is at risk,
prints the two commands it would have run, and makes no change. Use
`codemode --install-mcp --account u1 --force` when you accept rediscovering their tools.

`--force` is not a shrug. Before it writes, it takes the account's whole `mcp` object out of the
daemon and saves it beside the settings file as `settings.json.codemode-bak-<stamp>`. If the
discovery session fails, or if `execute_code` is not in the inventory afterwards, that snapshot
goes straight back and the run reports `rolledBack`. If it succeeds, the command waits for
Aside's migration to finish and then switches back on every server that was enabled before and
is disabled now, naming them in `restored`, and naming in `lostInventories` the servers whose
cached tools were dropped. Those caches are not rebuilt by hand: Aside rediscovers them on their
next session, and a cache written from a snapshot is a claim about a tool definition nobody
re-read. [Measured](evidence/mcp-force-restore-260918.md) against an unreachable probe server:
Aside disabled it 2.0 s into the run and the restore switched it back on at 3.6 s.

For a machine that already has other MCP servers, the older file-based path remains available.
From the package checkout, run
`node scripts/install-codemode.mjs install --account 0 --json` to write `settings.json`, then
open **Settings > Plugins & MCPs > MCPs**, select `aside-codemode`, and use **Refresh tools**.
Alternatively, restart the Aside daemon so it reloads `settings.json`, then let an ordinary
session run discovery. Confirm that `execute_code` is cached before starting a new work session.

You do not have to guess which state you are in. `codemode --doctor` reports one per Aside
account: `not-registered`, `registered-not-activated`, `activated`, or `stale-entry` when the
entry still points at an older installation. Anything but `activated` comes with the exact next
step. The CLI never writes Aside's inventory cache by hand; doing so would go stale when
the tool definition changes.

The agent then calls `mcp__aside-codemode__execute_code` directly. Its **2,043-byte** tool
description is always resident in every MCP session. That is less resident context than the CLI
route's 3,808-byte account block, one reason MCP is the first-class route.

The daemon starts the server with only six environment variables and with its own application
directory as cwd, so its PATH does not contain ripgrep. The server therefore resolves Aside's
bundled native binary directly: ripgrep 15.2.0 with PCRE2 on macOS and 15.1.0 with PCRE2 on
Windows. An explicit absolute `rgPath` or `CODEMODE_RG` still overrides that resolution.

### Route 2: CLI and account guidance

For this route, the CLI alone does nothing for an agent. Aside must be told the absolute
node/CLI pair and given the skill that explains the call shapes. One command names the account:

```sh
node scripts/install-codemode.mjs install --account 0 --json
```

It writes six files under that account root and one markered block inside its `AGENTS.md`:

```
codemode/cm.js                                     the batch helper an Aside REPL loads
codemode/catalog.json                              the action catalog
codemode/manifest.json                             what this install owns, by hash
skills/user/aside-codemode/SKILL.md                when to batch, how to read a result
skills/user/aside-codemode/references/*.md         call shapes, execution paths, Windows quoting
AGENTS.md  <!-- aside-codemode:start … end -->     the block read on every turn
```

The Route 2 payload owns only the files and markered block listed above. Credentials, sessions,
memory and other skills are not ours and are never opened. A file you edited is kept and named
in `preserved` rather than overwritten, `doctor` tells you when a machine is behind a release, and
`uninstall` removes only the files whose hashes still match and takes its block back out of
`AGENTS.md` while leaving the rest of that file alone.

Always pass `--account`. Without it the installer follows `accounts.json`'s `currentAccountId`,
which can change under you. Run it again after every upgrade: a release that changes only the
guidance still leaves an account stale, and `doctor` will say so.

Browsing needs no third step. It is on by default.

The agent reaches code mode with one `bash` call to that absolute node/CLI pair. The account
`AGENTS.md` block costs **3,808 bytes** of always-resident context; the **8,973-byte** installed
user skill is loaded on demand. This remains a supported route for hosts that do not attach MCP
servers and for people who prefer account guidance and a visible bash call.

### Remove the CLI-route artifacts after moving to MCP

Once native MCP is working, remove the Route 2 payload for one account with:

```sh
node scripts/install-codemode.mjs uninstall --account 0 --json
```

The uninstaller removes the managed `aside-codemode` marker block from `AGENTS.md`, the installed
user skill and its references, `codemode/cm.js`, `codemode/catalog.json`, and the manifest. For
payload files, it removes only those whose hashes still match; modified files stay behind and are
reported in `preserved`, while the manifest itself is removed after that ownership check. It
leaves the rest of `AGENTS.md`, credentials, sessions, memory, other
skills, `settings.json`, and every MCP setting or inventory untouched. Native MCP therefore stays
registered. The package installed by npm, the repository checkout, `codemode.config.json`,
user-modified files, and non-empty user directories also stay behind.

If you would rather ask Aside to clean up conversationally, paste this prompt:

```text
Remove the CLI-route artifacts for aside-codemode from account ~/.aside/u/<n>/. Remove only the managed aside-codemode marker block, from <!-- aside-codemode:start --> through <!-- aside-codemode:end -->, in AGENTS.md and leave everything outside that block unchanged. Remove the installed skills/user/aside-codemode skill and its references, codemode/cm.js, codemode/catalog.json, and codemode/manifest.json only when they belong to this install; preserve modified or user-owned files. Do not touch settings.json or any MCP configuration.
```

### Who uses the PATH command

Route 1 calls the configured MCP server directly and does not look up `codemode` on PATH. On
Route 2, the PATH command is for **you**, the operator. Aside agents call the absolute pair the
installer wrote into the AGENTS block (`process.execPath` plus this install's
`bin/codemode.mjs`); they must not look up `node` or `codemode` on PATH.

```sh
codemode --code "return (await search.files({ path: '/Users/me/proj', glob: '**/*.ts' })).length"
codemode --doctor
```

If `npm prefix -g` is not on PATH (Aside sets `NPM_CONFIG_PREFIX`, which wins over the default), install with an explicit prefix:

```sh
npm install -g --prefix=/opt/homebrew .
```

## Project cwd

Resolution order: `--cwd <abs>` > `CODEMODE_CWD` > `process.cwd()`. Relative guest paths resolve against that directory. Child agents should pass `--cwd` to the project they are editing. A missing `--cwd` flag is not an error; a `--cwd` with no directory is `{ok:false,error:"--cwd requires a directory path"}`.

## Guest API

Code is an async function body. `return` is the answer. The guest API does not expose `require`, `process`, `fetch`, or network tools. This is not a hostile-code security guarantee; see the trust model.

| Name | Role |
| --- | --- |
| `search.files` / `search.content` / `search.count` | ripgrep-backed list, content, pre-flight counts |
| `read_file({ path, offset?, limit? })` | Aside-shaped read. `offset` / `limit` are 1-indexed **lines**. Unpaged reads over 262144 bytes throw |
| `write_file({ file_path, content })` | Aside-shaped create-only (`wx`). Overwrite throws |
| `edit_file({ path, appendText?, edits })` | Unique `oldText` → `newText` on the original file |
| `apply_patch(text)` | Guest helper. Codex `*** Begin Patch` text → `write_file` / `edit_file`. Success `{}`. Not an AGENTS verb |
| `fs.readMany` / `grepFile` / `mkdir` / `stat` / `exists` / `list` | Compound helpers. `fs.read` / `fs.write` are deprecated byte / overwrite aliases |
| `actions.list` / `find` / `describe` / `check` | In-sandbox discovery |
| `browse.probe()` | Capability matrix measured against the installed Aside build: which page methods exist, which options are accepted-and-ignored, and why a request is refused |
| `browse.exec(job)` | A batch of URLs through ONE Aside REPL session. Returns `{ items, partial, leakedUrls }`; one failed URL never empties the others |
| `browse.captureMany(urls, { outDir, screenshot, ... })` | Batch capture. Screenshots come back as real files under `outDir`, each verified against the request — `clip` geometry is checked against the actual pixels rather than trusted |
| `browse.readText(url)` or `browse.readText({ url })` | Fetch-first read: HTML to markdown with no browser, falling back only when the fetched page measurably rendered no text. The body comes back as `text`, with `format` saying what it is (`markdown` from the fetch path, `text` from the browser's rendered body). Reports `source` and `fallbackReason` so you know which path answered |
| `browse.exec({ extract })` | Schema extraction in one `page.evaluate`: `{ field: 'css' }` or `{ selector, attr?, all? }`. Returns typed JSON plus a `missing[]` list, so absent is distinguishable from empty, and no snapshot tree is shipped |
| `browse.exec({ treeNodes })` / `browse.attach({ treeNodes })` | Off by default. With `treeNodes: true` the accessibility tree also arrives parsed as `snapshot.nodes`, one `{ depth, role, name, ref, attrs, line }` row per node, so hierarchical data is grouped by depth rather than by a regex over the string form |
| `api.batch(requests)` | Parallel API-first lookups. `youtube` and `itunes` are public no-key endpoints; `play` and `slack` refuse with `ENOTSUP` and the reason, because neither has an honest public path |
| `report.build({ items, outFile })` | Assembles a paged HTML report, prints it over an ephemeral loopback origin (`file://` is refused by Aside), and **verifies the real MediaBox**. `pdf({format:'A4'})` was measured to yield US Letter, so the size is proven rather than requested |
| `browse.searchMany(queries, { engine })` | N queries in parallel, URL-deduped, date-filtered. `youtube` works; `google` is callable but answers with a bot challenge, so it returns `EBLOCKED` with the URL to open rather than an empty result set; `duckduckgo` is the no-key default and gets the same challenge detection |
| `browse.downloadMedia(urls, { outDir })` | Original images by direct fetch. The **magic bytes gate the write**, so a block page claiming `image/png` is refused instead of landing on disk as a `.png` |
| `browse.watch(urls)` 