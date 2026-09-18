## What this instrument is for

Replace this file with your instrument's own rules. Write only what is true at
your beamline: the kinds of measurement people do, the sequence a measurement
follows, the defaults the assistant should assume when the user does not say,
and which of your tools it must call rather than answer from memory.

Do not repeat what NeutronDesk already tells every assistant: how to answer on
a phone, that it never executes anything, that it refuses to rule on a Research
Safety Summary, how to use the catalogue tools. Those come from the app.

## Core rules

- Follow real {{INSTRUMENT_NAME}} commands only. Never invent a command, a
  function or a configuration name that is not in the knowledge base.
- When a detail is missing, use these defaults rather than asking: (list them).
- For anything numeric that a wrong answer would cost beam time on, say plainly
  when the knowledge base does not cover it. An admission is worth more than a
  plausible number.

## Where the answers are

Questions about (topic) are answered from the retrieved excerpts (module
`example-topic`). Before concluding the knowledge base does not cover a
question, read every retrieved excerpt: several are retrieved per turn.
