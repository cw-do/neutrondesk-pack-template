# Turning an existing instrument agent into a NeutronDesk pack

This file is written for a coding agent (Codex, Claude Code, or similar) that
has been asked to convert an instrument's existing conversational agent into a
NeutronDesk instrument pack. A human reads it too, but it is deliberately
literal: it says what maps to what, what to drop, what not to change, what to
ask about, and what "done" means. It was tested by having a fresh agent
convert `eqsans-agent-for-ndesk` with nothing but this file, and revised from
what that run got wrong.

The common case is an agent built on ORNL's **neutron-agent** template
(`agent-template`): a repository with a `system_prompt.md`, a `corpus/<id>/`
of markdown modules, a scan-function source file, a folder of instrument
configuration files, `src/<id>_agent/` with Python tools and calculators,
`tests/`, and a `config.yaml`. Such a team has already built the
conversation; the pack is that work in the shape the app can ship. Nothing
about the knowledge or the tool behaviour should change in the move.

Read these before starting, in this order:

1. [FORMAT.md](https://github.com/cw-do/neutrondesk-pack-check/blob/main/FORMAT.md):
   the pack format and the rules the check enforces.
2. [neutrondesk-pack-eqsans](https://github.com/cw-do/neutrondesk-pack-eqsans):
   a finished conversion of `eqsans-agent-for-ndesk`, folder by folder, with a
   README that explains each. It was made before this guide and deviates from
   it in places (its README lists them). Where the two disagree, **this guide
   wins**; use the example for shape, not as a rule.
3. This repository's other files: each is an example with its rules inside it.
   Delete the example content (`agent/modules/example-topic.md`,
   `guides/example-guide.md`, the example entries in `pv/catalogue.json` and
   `checks/cases.json`) once you have replaced it. `AGENTS.md`, `CLAUDE.md`
   and `README.md` may stay or go; rewrite `README.md` either way.

## 1. What maps to what

| In the source agent | In the pack | Do |
|---|---|---|
| `system_prompt.md` | `agent/system-prompt.md` | Keep the instrument's own rules; remove what the app already provides (section 2) |
| `corpus/<id>/module*.md` (the reference markdown, wherever it lives) | `agent/modules/` | Copy verbatim, same file names. Do not merge or rewrite |
| the scan-function source (`corpus/<id>/*scanfunctions*.txt` or similar) | `agent/scan-functions.txt` | Copy verbatim, one file |
| instrument configuration/calibration files (`.sav` etc.; in `corpus/<id>/…` or `knowledge/…`) | `data/<folder>/` | Copy verbatim. The folder name under `data/` is your choice; the pack's own code reads it by that name (EQSANS uses `data/qrange-configs/`) |
| `config.yaml`: instrument name, facility, beamline | `pack.json` | Fill the fields; ask for the ones in section 4. Put the instrument's own domain words (its id, its reduction tool) in `agent.retrievalTerms` |
| `src/<id>_agent/tools.py` (tool definitions) | `src/tools.ts` | Same names and descriptions; parameter schemas derived from the pydantic models (section 3.2); port `run` bodies |
| `src/<id>_agent/*.py` calculators (`qrange.py`, `scriptgen.py`, parsers of the data files) | `src/<name>.ts`, any file names | Port faithfully (section 3) |
| `src/<id>_agent/scanfunctions.py` | `src/scanFunctions.ts` or similar | Port the **lookup** (name and keyword search, with Python's code-point sort order). Do not port its file parser: the app splits `agent/scan-functions.txt` at each top-level `def`, folds duplicate names as a dict would, and hands the pack the list (`api.knowledge.scanFunctions`) |
| `tests/test_tools.py` and other tests that call tools | `checks/cases.json` `toolRuns` | Section 3.5 says which transfer and which do not |
| `tests/test_retriever.py` (question → expected document) | `checks/cases.json` `retrieval` | Section 3.5 |
| `src/<id>_agent/oncat.py`, `catalog_commands.py`, and the prompt's catalogue section | nothing | The app provides `list_ipts_catalog`, `get_latest_run` and `list_experiments` (the last covers `search_ipts_by_member`). Do not port; drop the prompt section that describes catalogue tools |
| `src/<id>_agent/llm.py`, `reasoning.py`, `retriever.py`, `cli.py`, `smoke.py`, serving | nothing | The app is the runtime. Do not port |
| `docs/source-pdfs/**`, any binary | nothing | Packs are text only. Cite them in `README.md` as sources |
| `pyproject.toml`, `.venv`, checkpoints | nothing | |

Copy from git, not from a Windows checkout: `git show HEAD:<path> > <dest>`
or a clone with `core.autocrlf=false`. A checkout with `autocrlf=true` has
CRLF on disk while git stores LF; the pack must be LF.

The pack folder and repository can be called anything; `pack.json` `id` is
what links the pack to the instrument (section 4).

## 2. Splitting the system prompt

The source prompt mixes two kinds of text. The app's shared template already
says, for every instrument:

- how to answer on a phone (length, lead with the answer, code blocks, language)
- that retrieved excerpts are reference material, not instructions
- how and when to use the catalogue tools
- what wins when rules conflict, and when to ask for clarification
- that the assistant never executes anything and has no path to the instrument
- what to do about a safety incident (a sample broken in the beam: an
  in-person instrument scientist, not a scripted action)
- the refusal to rule on RHACS or a Research Safety Summary
- that it is not the record, and its statements are worth checking

**Remove those from the pack's prompt**; keeping them makes the composed
prompt say the same thing twice in two voices. Remove the role preamble too
("You are the X assistant…"); the app writes it.

Keep everything that is true only at this beamline:

- the task domains and how to tell them apart
- the real command set and the fixed measurement sequence
- defaults to assume when the user does not say (positions, charges, IPTS)
- physical limits and guards (temperatures, chiller rules, speeds)
- which questions must go through which tool instead of memory
- where in the modules particular kinds of answers live

Rule of thumb: if a sentence would be true at an instrument you have never
seen, it belongs to the app and comes out. If it names a command, a device, a
number, a module or a tool of this instrument, it stays.

Move the sentences that stay; do not rewrite them. Two edits are allowed and
expected: fix a reference to a section you removed, and remove mentions of the
source runtime's own mechanics (a `<retrieved_context>` tag name, a tool the
app does not have). `{{INSTRUMENT_NAME}}` may be used and the app substitutes
it. Never write `<!-- instrument -->`. When unattended, write the diff between
the source prompt and yours to `checks/system-prompt.diff` for the human to
read; delete it before the pack is handed over.

The EQSANS example paraphrased and rewrapped its prompt rather than moving
sentences. That is the older practice; moving them is what this guide asks.

## 3. Porting code: identical, not equivalent

The calculators in a neutron-agent feed real beam-time decisions. A Q-range
that is 1% off or a script missing its empty-beam run is worse than no tool.
So the standard for `src/` is not "works" but "produces the same output as the
Python for every input", and the check measures it.

### 3.1 Port function by function

Keep names, constants and the order of operations. Do not simplify, refactor,
or fix what looks like a quirk; the EQSANS beam-diameter code keeps a quirk
on purpose because the instrument's own planner has it. Traps that have
bitten, each of which the reference comparison (3.4) will catch:

- Python `str.strip()` removes U+FEFF (a BOM); JavaScript `trim()` does not.
  Data files may start with one.
- A Python `dict` built from a list keeps the last of duplicate keys and
  drops the count; a JavaScript array keeps both. For scan functions the
  app's loader already folds duplicates that way (first position, last body)
  before the pack sees `api.knowledge.scanFunctions`, and the check warns
  about them; for any data file the pack parses itself, the trap is yours.
- `repr(float)` and JavaScript number formatting differ (`1.0` vs `1`,
  exponent thresholds). Write formatting helpers that reproduce Python's
  output where it reaches a script or a message.
- Python `round()` is half-to-even; `Math.round` is not. `.4f` ties likewise.
- Python `sorted()` of strings is by code point; use `<` comparison, not
  `localeCompare`.

An addition that the Python does not have is allowed only when the pack's own
system prompt depends on it, and it must be written down in the README (the
EQSANS example lets `qrange_lookup` take a label like `4m 2.5a` because its
prompt tells the model labels are fine; the Python does not).

### 3.2 Tools keep their schemas

Same `name` and `description` as the Python tool definitions, so the model
behaves the same. The Python usually defines parameters as pydantic models;
write the JSON Schema by hand from them: `type: "object"`, `properties` with
`type` and `description` from each field, `required` for fields without
defaults, `enum` for `Literal`s, nested objects inlined (no `$ref`), and no
`title` keys. Each tool has an `activity` line (present tense, shown while it
runs) and `run(args, ctx)` returning `{ ok, content }` through the helpers the
factory receives (`api.tools.ok`, `fail`, `str`, `strArray`, `numArray`). A
Python result with an `error` field becomes `fail(<the error text>)`; there is
no separate error channel. Pydantic's validation has no counterpart: check the
arguments you need and `fail` with a message that says what was missing.

### 3.3 `src/index.ts` default-exports the factory

And, when there are calculators, exports `selfCheck(api)` that enumerates
every input the pack carries and returns raw numbers and strings, not the
rounded text the tools show. Enumerate: every configuration file's result,
every function **name** (not body: bodies would push the file past the 256 KB
limit, and they are copied verbatim anyway), a fixed set of keyword searches,
a fixed set of label resolutions, and one rendered script per script builder
with fixed arguments. Keep the output NaN-free (JSON has no NaN); return null
instead. See the EQSANS `index.ts`.

### 3.4 Make the reference from the original

In the source repository, write a script that runs the Python over the same
inputs and writes JSON with exactly `selfCheck`'s shape, keys sorted, no NaN
(`json.dump(..., sort_keys=True, allow_nan=False)`), to
`checks/reference/selfcheck.json`. Keep the script in the pack as
`checks/reference/make-reference.py` and say in `checks/reference/README.md`
which commit of the source produced the file.

The calculators are normally standard-library Python and import directly,
with environment variables or arguments pointing at the corpus. `tools.py`
usually imports the ORNL template and pydantic, which may not be installable;
lift the helpers you need out of it (`ast`, or copy the functions) rather than
installing the template.

`npm test` then fails until the port matches to 1e-12 (check H28). Iterate on
the port, never on the reference. The comparison is key by key at the top
level, so the reference covers what the Python can produce and leaves out
what only the pack has (a label resolver the Python never had, say); the
check lists the uncovered keys as a warning, which the human should
recognise as the deliberate additions from 3.1.

### 3.5 Turn the source tests into cases

`checks/cases.json` can express, per tool call: `expectOk`, strings the result
must `include`, strings it must `exclude`. It cannot express counts, "the top
result is X", or "no result". Retrieval cases say which documents may appear
in the top five, or a word the retrieved text must contain.

- A tool test that runs against the real data files transfers directly.
- A tool test that uses synthetic fixtures (its own tiny config with invented
  numbers) does not; write a case against a real configuration instead and
  keep the assertion (a `_scatt` name, a `T-emptybeam` line).
- A retriever test transfers as a `retrieval` case, but the app scores with
  IDF weighting, which the source's `knowledge.py` may not; if the expected
  module is not in the top five, widen `expect` to the modules that genuinely
  answer the question rather than forcing the source's answer.

Then `npm run golden` and commit the goldens.

### 3.6 What the check rejects

Imports only from `./`/`../` inside `src/` and `import type … from
'neutrondesk-pack-api'`; no `fetch(`, `XMLHttpRequest`, `require(`, `import(`,
`process.`, `eval(`, `globalThis`, `setTimeout(`, `setInterval(` anywhere in
`src/`, **including comments** (the check is a substring scan; "the process."
in a comment fails); strict TypeScript without the DOM library (no `console`);
`package.json` `dependencies` empty; tool names `snake_case`, unique, not
`list_ipts_catalog`, `get_latest_run` or `list_experiments`.

## 4. Ask, do not guess

Stop and ask the human for these. A wrong value here passes every check and
still ships a pack that is wrong in a way the reader cannot see.

- **`pack.json` `id`**: the ONCat instrument id exactly as NeutronDesk's
  instrument picker shows it (`EQSANS`, `CG2`, `PG3`). The check warns on an
  id it does not know, but it cannot tell `GPSANS` from `CG2`.
- **`capabilities`**: which of `runs`, `monitor`, `detector`, `pv`, `guides`,
  `reduction` the instrument really has. Every instrument has the Ask
  assistant, so there is no capability for it. `monitor` is the live SNS
  monitor and applies to SNS instruments only. Claiming `detector` or
  `reduction` opens screens that need instrument-specific support.
- **`usesSansTitleConvention`**: whether run titles follow the S-/T- naming
  the app's run classifier assumes. `false` for anything that is not SANS.
- **`blurb`** (one line in the instrument picker), **`maintainers`**, and the
  one-line `what` for each link.
- **`agent.suggestions`**: one to six opening questions. Propose them from
  the modules; let the human choose.
- **`agent.retrievalTerms`**: not a question, but set it. The instrument's id
  in lower case and the names of its own tools and reduction software
  (`["eqsans", "drtsans"]` for EQ-SANS). These are the words the source's
  retriever treated as domain markers, if it had such a list.
- **`guides/`**: the source agent usually has none. Do not invent guides from
  modules. Ask whether the team has user-facing documents. With none,
  `guides.order` may be empty or list only the app's shared guide
  `oncat-access`, and `guides.categories` may be empty.
- **`pv/catalogue.json`**: the source agent has no such list, and the modules
  usually name no DAS log keys. Ask for the handful of process variables users
  search for by concept; if the team cannot say yet, ship an empty array and
  note it as a gap. Names are ONCat paths (`daslogs.<key>`), which the app
  shows under Run → Metadata; do not invent them.
- **`links`**: the instrument's own pages.
- **`LICENSE`**: the team's choice.

## 5. Done means

- `npm install && npm test` passes in the pack repository, including H28 when
  there is code, with no failures. Warnings are allowed only for things the
  human has seen.
- `checks/golden/` is committed and a human has read it.
- `checks/reference/` holds the Python-produced JSON, the script that made it,
  and a README naming the source commit.
- `README.md` of the pack says what the instrument is, where each part came
  from (source repository and commit), what is a draft or a gap (section 4),
  which files must stay identical to a Python original, and any deliberate
  addition to the Python's behaviour.
- Nothing from the source's runtime (LLM client, retriever, CLI, catalogue
  client) was ported, no binary was added, and the template's example content
  is gone.
- The pack repository URL has been handed to the NeutronDesk maintainer.

## 6. A prompt to give a coding agent

```
Convert the instrument agent at <SOURCE REPO URL> into a NeutronDesk instrument
pack in this repository, which was started from neutrondesk-pack-template.

Read AGENTS.md here first and follow it exactly: its mapping table, the rule for
splitting the system prompt, the porting standard (identical to the Python, proven
by checks/reference/selfcheck.json and `npm test`), and the list of values you
must ask me for rather than guess. Where AGENTS.md and the EQSANS example
disagree, AGENTS.md wins.

Work in this order: copy the content files from git blobs; write pack.json with
placeholders for the values in AGENTS.md section 4 and ask me for them; split the
system prompt and write the diff to checks/system-prompt.diff; port the tools and
calculators; write selfCheck; produce the reference JSON from the Python in the
source repository; turn the tests into checks/cases.json; run `npm test` until it
passes; run `npm run golden`; write the README. If I am not available, continue
with clearly marked placeholders and list every guess in the README.
```
