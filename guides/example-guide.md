---
id: example-guide
title: Example guide
category: experiment
summary: One sentence the Guides screen shows under the title; replace it, and rename this file to match its id.
updated: 2026-09-18
source: where this text came from, e.g. the instrument's own documentation page
---
# Example guide

A guide is a document users read on the app's Guides screen. It is also
retrieved by the assistant alongside the modules, so writing one does both.

The front matter above is required, every key. `id` must equal the file name
without `.md`, and every guide must be listed in `pack.json` under
`guides.order`, in the order the screen should show them. `category` must be
one of the categories the pack declares in `guides.categories`.

## What to put here

- Steps someone follows, numbered.
- Commands in fenced code blocks, so they can be copied from the phone.
- Where the current version of this information lives, if it is a web page
  the facility maintains. A link on the Guides screen beats a copy that goes
  stale; add such pages to `pack.json` `links` too.
