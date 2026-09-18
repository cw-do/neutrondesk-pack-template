Example topic: replace with one focused topic
==============================================

A module is one topic the assistant can be asked about, written as reference
material. Whole files are scored against each question by the words they share
with it, so keep one topic per file: a file that covers two topics competes
with itself and reaches neither question well.

The first line above, underlined with `===`, is the module's title. Files sort
by the first number in their name, so `module2.md` comes before `module10.md`;
names without numbers sort alphabetically.

Write for someone standing at the instrument reading a phone. Say what the
thing is, what the defaults are, and what goes wrong when they are ignored.
Commands, function signatures and file paths go in fenced code blocks so the
assistant can quote them exactly.

```python
# An example command block. The assistant quotes these verbatim.
example_function(sample_position=2, proton_charge=1.0)
```

Anything that is not `.md` in this folder is ignored, not partially read.
