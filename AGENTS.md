# AGENTS.md

Context for AI coding assistants using this map of the Arturia KeyLab mk3's
SysEx.

- `keylab-mk3-sysex.md` is the reference. Every claim carries a marker:
  `[measured]` on hardware, `[code]` read from Arturia's script, `[unverified]`,
  `[guess]`, and `[rejected]` for things proven wrong. Rank them in that order,
  and do not build on a `[rejected]` line.
- A working Live control surface built on this map, with the rules learned along
  the way and a list of approaches that have already failed, is in
  **[KeyLabLive](https://github.com/saxofonbrian/KeyLabLive)**. Read its
  `AGENTS.md` before writing code.
- When something does not behave as documented: record Arturia's own script
  with a MIDI monitor and compare the bytes with yours before forming a theory.
