# Turning an existing instrument agent into a NeutronDesk pack

This file is written for a coding agent (Codex, Claude Code, or similar) that
has been asked to convert an instrument's existing conversational agent into a
NeutronDesk instrument pack. A human reads it too, but it is deliberately
literal: it says what maps to what, what to drop, what not to change, what to
ask about, and what "done" means.

The common case is an agent built on ORNL's **neutron-agent** template
(`agent-template`): a repository with a `system_prompt.md`, a `corpus/<id>/`
of markdown modules and a scan-function source file, a `knowledge/` folder of
instrument configuration files, `src/<id>_agent/` with Python tools and
calculators, `tests/`, and a `config.yaml`. Such a team has already built the
conversation; the pack is that work in the shape the app can ship. Nothing
about the knowledge or the tool behaviour should change in the move.

Read these before starting, in this order:

1. [FORMAT.md](https://github.com/cw-do/neutrondesk-pack-check/blob/main/FORMAT.md):
   the pack format and the rules the check enforces.
2. [neutrondesk-pack-eqsans](https://github.com/cw-do/neutrondesk-pack-eqsans):
   the finished conversion of `eqsans-agent-for-ndesk`, folder by folder, with
   a README that explains each. When in doubt, do what it does.
3. This repository's other files: each is an example with its rules inside it.

## 1. What maps to what

| In the source agent | In the pack | Do |
|---|---|---|
| `system_prompt.md` | `agent/system-prompt.md` | Keep the instrument's own rules; remove what the app already provides (section 2) |
| `corpus/<id>/module*.md` (or wherever the reference markdown lives) | `agent/modules/` | Copy verbatim. Keep file names. Do not merge or rewrite |
| `corpus/<id>/*scanfunctions*.txt` (the scan-function source) | `agent/scan-functions.txt` | Copy verbatim, one file |
| `knowledge/**` configuration or calibration files (e.g. `.sav`) | `data/<same folder name>/` | Copy verbatim. `data/` is handed to the pack's code as strings |
| `config.yaml`: instrument name, facility, beamline | `pack.json` | Fill the fields; ask for the ones in section 4 |
| `src/<id>_agent/tools.py` (tool definitions) | `src/tools.ts` | Same names, descriptions and parameter schemas; port `run` bodies |
| `src/<id>_agent/*.py` calculators (`qrange.py`, `scriptgen.py`, `scanfunctions.py`, parsers) | `src/<same name>.ts` | Port faithfully (section 3) |
| `tests/test_tools.py`, `tests/test_*.py` cases that call tools | `checks/cases.json` `toolRuns` | Each test that calls a tool with arguments and asserts on output becomes one case |
| `tests/test_retriever.py` cases (question → expected document) | `checks/cases.json` `retrieval` | Each becomes one case |
| `src/<id>_agent/oncat.py`, `catalog_commands.py` | nothing | The app provides the catalogue tools (`list_ipts_catalog`, `get_latest_run`, `list_experiments`). Do not port |
| `src/<id>_agent/llm.py`, `reasoning.py`, `retriever.py`, `cli.py`, `smoke.py`, `serve` | nothing | The app is the runtime. Do not port |
| `docs/source-pdfs/**`, any binary | nothing | Packs are text only. Cite them in `README.md` as sources |
| `pyproject.toml`, `.venv`, `data/checkpoints*` | nothing | |

Names: the pack folder and repository can be called anything; `pack.json`
`id` is what links the pack to the instrument (section 4).

## 2. Splitting the system prompt

The source prompt mixes two kinds of text. The app's shared template already
says, for every instrument: how to answer on a phone, that the assistant never
executes anything, that it refuses to rule on a Research Safety Summary or
RHACS, how to use the catalogue tools, what wins when rules conflict, and when
to ask for clarification. **Remove those from the pack's prompt**; keeping
them makes the composed prompt say the same thing twice in two voices.

Keep everything that is true only at this beamline:

- the task domains and how to tell them apart
- the real command set and the fixed measurement sequence
- defaults to assume when the user does not say (positions, charges, IPTS)
- physical limits and guards (temperatures, chiller rules, speeds)
- which questions must go through which tool instead of memory
- where in the modules particular kinds of answers live

Rule of thumb: if a sentence would be true at an instrument you have never
seen, it belongs to the app and comes out. If it names a command, a device, a
number, a module or a tool of this instrument, it stays. Do not paraphrase
what stays; move the sentences. `{{INSTRUMENT_NAME}}` may be used and the app
substitutes it. Never write `<!-- instrument -->`.

Compare your result with `agent/system-prompt.md` in the EQSANS pack: six
sections, all domain, none of the generic behaviour.

## 3. Porting code: identical, not equivalent

The calculators in a neutron-agent feed real beam-time decisions. A Q-range
that is 1% off or a script missing its empty-beam run is worse than no tool.
So the standard for `src/` is not "works" but "produces the same output as the
Python for every input", and the check measures it.

1. **Port function by function, keeping names, constants and order of
   operations.** Do not simplify, refactor, or fix what looks like a quirk;
   the EQSANS beam-diameter code keeps a quirk on purpose because the
   instrument's own planner has it. If the Python formats a float as `1.0`,
   the TypeScript writes `1.0`.
2. **Tools keep their schemas.** Same `name`, `description`, `parameters` and
   `required` as the Python tool definitions, so the model behaves the same.
   Each tool has an `activity` line (present tense, shown while it runs) and a
   `run(args, ctx)` returning `{ ok, content }` through the helpers the factory
   receives (`api.tools.ok`, `fail`, `str`, `strArray`, `numArray`).
3. **`src/index.ts` default-exports the factory** and, when there are
   calculators, exports `selfCheck(api)` that enumerates every input the pack
   carries (every configuration file, every function name, a fixed set of
   keyword searches and label resolutions) and returns the raw numbers and
   strings, not the rounded text the tools show. See the EQSANS `index.ts`.
4. **Make the reference from the original.** In the source repository, write
   a small script (keep it as `checks/reference/make-reference.py` in the
   pack) that runs the Python over the same inputs and writes JSON with
   exactly `selfCheck`'s shape, keys sorted, to `checks/reference/selfcheck.json`.
   `npm test` then fails until the port matches to 1e-12 (check H28). Iterate
   on the port, never on the reference. Say in `checks/reference/README.md`
   which commit of the source produced it.
5. **Turn the source tests into cases.** Every test that calls a tool becomes
   a `toolRuns` entry with `includes`/`excludes`/`expectOk`; every retriever
   test becomes a `retrieval` entry. Then `npm run golden` and commit the
   goldens.

Constraints the check enforces on `src/` (and will reject): imports only from
`./`/`../` inside `src/` and `import type ... from 'neutrondesk-pack-api'`; no
`fetch`, `XMLHttpRequest`, `require`, dynamic `import`, `process`, `eval`,
`globalThis`, `setTimeout`, `setInterval`; strict TypeScript without the DOM
library (no `console`); `package.json` `dependencies` empty; tool names
`snake_case`, unique, not `list_ipts_catalog`, `get_latest_run` or
`list_experiments`.

## 4. Ask, do not guess

Stop and ask the human for these. A wrong value here passes every check and
still ships a pack that is wrong in a way the reader cannot see.

- **`pack.json` `id`**: the ONCat instrument id exactly as NeutronDesk's
  instrument picker shows it (`EQSANS`, `CG2`, `PG3`). The check warns on an
  id it does not know, but it cannot tell `GPSANS` from `CG2`.
- **`capabilities`**: which of `runs`, `monitor`, `detector`, `pv`, `guides`,
  `agent`, `reduction` the instrument really has. Claiming `detector` or
  `reduction` opens screens that need instrument-specific support.
- **`usesSansTitleConvention`**: whether run titles follow the S-/T- naming
  the app's run classifier assumes. `false` for anything that is not SANS.
- **`agent.suggestions`**: one to six opening questions. Propose them from the
  modules; let the human choose.
- **`guides/`**: the source agent usually has none. Do not invent guides from
  modules. Ask whether the team has user-facing documents to add.
- **`pv/catalogue.json`**: the source agent has no such list. Propose entries
  only for process variables the modules actually name, mark the file as a
  draft in `README.md`, and ask the team to confirm units and names.
- **`links`**: the instrument's own pages. Ask.
- **`LICENSE`**: the team's choice.

## 5. Done means

- `npm install && npm test` passes in the pack repository, including H28 if
  there is code, with no failures. Warnings are allowed only for things the
  human has seen.
- `checks/golden/` is committed and a human has read it.
- `README.md` of the pack says what the instrument is, where each part came
  from (source repository and commit), what is a draft (section 4), and which
  files must stay identical to a Python original.
- Nothing from the source's runtime (LLM client, retriever, CLI, catalogue
  client) was ported, and no binary was added.
- The pack repository URL has been handed to the NeutronDesk maintainer.

## 6. A prompt to give a coding agent

```
Convert the instrument agent at <SOURCE REPO URL> into a NeutronDesk instrument
pack in this repository, which was started from neutrondesk-pack-template.

Read AGENTS.md here first and follow it exactly: its mapping table, the rule for
splitting the system prompt, the porting standard (identical to the Python, proven
by checks/reference/selfcheck.json and `npm test`), and the list of values you
must ask me for rather than guess. Use https://github.com/cw-do/neutrondesk-pack-eqsans
as the worked example of the result.

Work in this order: copy the content files; write pack.json with placeholders for
the values in AGENTS.md section 4 and ask me for them; split the system prompt;
port the tools and calculators; write selfCheck; produce the reference JSON from
the Python in the source repository; turn the tests into checks/cases.json; run
`npm test` until it passes; run `npm run golden`; write the README. Stop and show
me the diff between the source system prompt and agent/system-prompt.md before
moving on to the code.
```
