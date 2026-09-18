# For coding agents

Read `AGENTS.md` first. It is the guide for turning an existing instrument
agent (a neutron-agent based repository) into this pack: what maps to what,
what to drop, the standard for porting code, and the values to ask a human
for rather than guess. `README.md` covers starting from nothing.

`npm test` is the arbiter. Do not edit anything under `node_modules/`, and do
not regenerate `checks/golden/` without showing the diff to a human.
