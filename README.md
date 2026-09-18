# NeutronDesk instrument pack template

The starting point for an instrument pack: everything one instrument brings to
[NeutronDesk](https://github.com/cw-do/neutrondesk) — the assistant's rules and
reference knowledge, the guide library, the friendly names for its process
variables, links, and optionally code for calculations the assistant should do
deterministically. The instrument team owns the pack; the app vendors it at
build time and never fetches it while running.

**The full format is [FORMAT.md](https://github.com/cw-do/neutrondesk-pack-check/blob/main/FORMAT.md)**,
published with the check tool. Read its "Making a pack, step by step" first.

## Two ways in

**You already have an instrument agent** (one built on ORNL's neutron-agent
template, with a system prompt, a corpus of modules, scan functions, Python
tools and tests). Then the pack is that work in the shape the app can ship,
and most of the move is mechanical. [AGENTS.md](./AGENTS.md) is the
conversion guide, written so a coding agent can follow it: the mapping table,
what to drop, the standard for porting code (identical to the Python, proven
by a reference file and `npm test`), and the values it must ask you for. It
ends with a prompt to hand to Codex or Claude Code together with your source
repository's URL. [neutrondesk-pack-eqsans](https://github.com/cw-do/neutrondesk-pack-eqsans)
is the finished conversion of `eqsans-agent-for-ndesk`.

**You are starting from nothing.** Follow "Start here" below; every example
file in this repository has its rules written inside it.

## Start here

```bash
git clone https://github.com/cw-do/neutrondesk-pack-template.git neutrondesk-pack-<instrument>
cd neutrondesk-pack-<instrument>
rm -rf .git && git init          # your own history, not the template's
npm install                      # brings in neutrondesk-pack-check
npm test                         # the example pack passes, with one warning about its id
```

Then replace, in this order:

1. `pack.json` — the `id` is the instrument's ONCat id exactly as the app's
   instrument picker shows it (`EQSANS`, `CG2`, `PG3`). It is the only thing
   that links the pack to the instrument. Then every other field.
2. `agent/system-prompt.md` — your beamline's rules only.
3. `agent/modules/*.md` — one topic per file. Delete `example-topic.md`.
4. `guides/*.md` — front matter required; list each in `pack.json`
   `guides.order`. Delete `example-guide.md`.
5. `pv/catalogue.json` — the process variables people search for by concept.
6. `checks/cases.json` — questions and the module they should reach.
7. `package.json` — the `name`; keep `dependencies` empty.
8. `README.md` — this file: what is specific to your instrument.
9. `LICENSE` — yours to choose. The EQ-SANS pack keeps all rights reserved;
   this template is MIT so you can copy it freely.

Run `npm test` as you go. When it passes, run `npm run golden`, look at what
appeared under `checks/golden/`, and commit it. Then hand the repository URL
to the NeutronDesk maintainer, who adds one line to the app's `packs.json`,
pulls, and ships the pack in the next build.

## Code packs

Most instruments need no code. If the assistant should compute something
exactly (a Q-range, a measurement script), add `src/index.ts` that
default-exports a factory; see FORMAT.md, and
[neutrondesk-pack-eqsans](https://github.com/cw-do/neutrondesk-pack-eqsans)
for a complete example with tools, cases and goldens.

## Rules the check enforces

UTF-8 and LF everywhere (`.gitattributes` is set for that), 2 MB total,
256 KB per file, no binaries, no `dependencies`, ids and categories from the
app's vocabularies, and for code: only relative imports plus types from
`neutrondesk-pack-api`, no network, compiles under strict TypeScript.
