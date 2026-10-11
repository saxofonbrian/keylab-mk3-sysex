# Arturia KeyLab mk3 — screen SysEx, CC maps and LED IDs

Reverse-engineered documentation of the Arturia KeyLab 61/88 mk3 screen
protocol, plus a working Ableton Live proof of concept that uses it.

There is no public documentation of this protocol. Arturia does not publish it,
and the one guide that exists online covers the KeyLab **Essential** mk3, which
uses a different structure entirely and does not transfer. Everything here was
measured on hardware, read out of Arturia's own `config.lua`, or logged off the
wire.

## [`keylab-mk3-sysex.md`](keylab-mk3-sysex.md) — the reference

The main content of this repository. Roughly 900 lines covering:

- **Both screen families.** `00 02 04` (DAW mode, MainStage script) and
  `00 01 04` (Arturia mode, Analog Lab) — packet layouts, field IDs, and the
  fact that they ignore each other.
- **Three list widgets** and what each one costs you in visible lines, plus how
  the cursor number maps to a screen line (it is not the line number).
- **MIDI pauses under a fader readout.** With a list on screen, each changing
  readout makes the keyboard stop sending MIDI for ~60 ms in every ~100. The
  list title `0x14` (one column) avoids it. Item fields (icons, text colour)
  are documented too.
- **Complete CC maps** for DAW mode and Arturia mode: screen buttons,
  transport row, jog wheel, faders, encoders, and the separate touch CCs.
- **LED IDs** for the transport row and the twelve pads, with the source of
  each one marked.
- **Pad banks** and the `21 03 40` configuration family where the User-mode pad
  colours actually live.
- **Dead ends**, listed explicitly — a dozen things that look like they should
  work and do not, so nobody has to measure them twice.

Claims are marked with their provenance where it matters: **[measured]** on
hardware, **[code]** used in production and works in daily use, but was never
isolated and varied, **[unverified]** read out of code or a log without being
tested, **[guess]** derived from a pattern rather than observed, **[rejected]**
tried and found wrong. Unmarked prose is explanation, not evidence. Check the
marker before you build on something.

## [`POC Project.zip`](POC%20Project.zip) — the proof of concept

An Ableton Live set and a Max for Live device that put the protocol to work.
Setup instructions are below.

---

# Proof of concept

A minimal Max for Live device that drives the KeyLab screen from Ableton Live:

- **Bottom row of screen buttons** arms and un-arms the first four tracks.
  The label is the track's own name, the colour shows arm state.
- **Centre of the screen** shows the Arrangement locators as a 7-line list.
- **Jog wheel** moves the cursor, **Play** jumps to the selected locator and
  starts, **Stop** stops.
- Playback **stops automatically at the next locator** and the cursor moves on,
  ready for the next song.

Everything lives in one device on one track: `keylab_poc.js` plus a Max patch
of seven objects. Nothing else is required — no control surface script, no
companion devices, no other software.

Tested on a KeyLab 61 mk3 with Live 12 on macOS. The SysEx is identical on the
88; only the MIDI port name differs.

## What you get

Download `POC Project.zip` and unpack it. Inside:

```
POC.als
M4L/
  POC.amxd
  keylab_poc.js
```

`M4L` must stay next to `POC.als`. The device looks for `keylab_poc.js`
alongside itself; if Max cannot find it, the `js` object comes up broken and
nothing reaches the screen. Check the Max console — a missing script is
reported there.

## 1. Put the KeyLab in DAW mode

**This is the step that decides whether anything appears on the screen at all.**

The keyboard has two independent worlds: Arturia mode (what Analog Lab uses)
and DAW mode. They use different SysEx families and different CC numbers, and
packets from one are ignored while the other's page is displayed. This POC
speaks DAW mode only.

Switch with the physical keys: **Prog + screen button 6**. (Prog + screen
button 5 goes back to Arturia mode.)

There is no SysEx that changes the mode. The device does send a *connect*
handshake — `F0 00 20 6B 7F 42 00 02 05 01 F7`, sent on every init — but that
only works when the unit is already in DAW mode. It is a handshake, not a mode
switch.

How to tell you are in DAW mode: the screen buttons send CC 45-52 (top row
45-48, bottom row 49-52). In Arturia mode the same buttons send CC 56-63.

## 2. Turn off the Arturia control surface in Live

**Preferences → Link, Tempo & MIDI → MIDI.**

In the Control Surface table, every row that mentions the KeyLab must be set to
**None**:

![Control Surface set to None](docs/control-surface-none.png)

Arturia's own control surface script writes track names to the same four
buttons this POC writes to, and it also sends a *disconnect* when you switch it
off. Leave it enabled and the two fight over the screen; you will see a mix of
both and conclude your own packets work when they do not.

In the **MIDI Ports** list below, for the KeyLab's **DAW** port:

|                          | Track  | Sync | Remote |
| ------------------------ | ------ | ---- | ------ |
| Input: KeyLab mk3 (DAW)  | **On** | Off  | Off    |
| Output: KeyLab mk3 (DAW) | **On** | Off  | Off    |

Track must be on for *both* directions. Input and output are separate settings,
and it is entirely possible to have buttons working while nothing reaches the
screen.

Remote off matters too, though only for controls you have mapped: with Remote
on, any CC that is bound in Live's MIDI Map Mode is consumed by Live and never
reaches the device. Unmapped CCs pass through either way. Leaving it off keeps
the question from arising.

Note there are two ports per keyboard. The plain one carries the keys; the DAW
one carries the screen, the transport buttons and the jog wheel. Their names
differ by a few characters. Pick the DAW port in every dropdown.

## 3. Open POC.als and check the control track

The set holds four instrument tracks, four named locators in the Arrangement,
and a fifth track — **M4L** — carrying nothing but the device. Check three
things on that track before anything else:

- **MIDI From:** KeyLab mk3 (DAW), All Channels
- **MIDI To:** KeyLab mk3 (DAW), Ch. 1
- **Track activator on** (the number box at the bottom lit, not grey)

Routing is stored by port name, so if your keyboard is not the same model the
set was saved with — a 61 where it says 88, or the other way round — both
dropdowns will come up empty and have to be set by hand. Everything else in the
device still behaves as if it works, which makes this an easy one to miss.

Monitor is already set to **In**. It has to be: on Auto, incoming MIDI is gated
by arm state and the device never sees it.

The track activator matters for the same reason. A muted MIDI track sends
nothing to its MIDI To output. The script still runs, the console still reports
success, arming still works through the Live API — and not one byte leaves
Live.

Then make it your own: the button labels are the track names, so renaming a
track relabels its button, and the list is simply the Arrangement locators, so
add, rename or move them freely. Press Init afterwards.

## 4. Run it

The device inits itself one second after the set loads. You should see four
track names along the bottom of the screen and the locator names in the middle.

Three ways to re-init:

- **CC44** — the Redo button on the keyboard
- Click **[init(** in the patch — with the patch **locked** (Cmd+E); clicking a
  message box in edit mode only selects it
- Reload the set

Init is not optional after editing `keylab_poc.js`. Autowatch reloads the
script without `live.thisdevice` firing, so `init()` never runs, nothing works,
and the Max console says nothing at all.

**[connect(** sends the DAW handshake on its own — useful when the keyboard was
switched on after Live.

## When the screen stays blank

Work down this list. It is ordered by how often each one is the answer.

1. **Is the M4L track's activator on?** Muted track, no MIDI out.
2. **Is the Arturia control surface set to None?** Also check whether it sent a
   disconnect on its way out — re-run init afterwards.
3. **Is the keyboard in DAW mode?** Prog + screen button 6.
4. **Is MIDI To the DAW port, and is Track on for the output?** Input and
   output are separate; working buttons prove nothing about the output path.
5. **Does the Max console print `keylab_poc: ready — 4 tracks, N locators`?**
   No line means init never ran, or Max could not find the `.js` file — check
   that the `M4L` folder travelled with the `.als`.
6. **Does a MIDI monitor see the packets?** Init sends seven packets plus one
   per locator: one connect, four button packets, one list title, one cursor,
   and one item per locator. If they are on the wire and the screen is still
   blank, the keyboard is not in DAW mode. If they are not, the fault is
   between the `js` object and the port.

The useful principle underneath: the console tells you whether the code ran, a
MIDI monitor tells you whether the bytes left the machine, and only the two
together tell you where the fault is. One of them alone will mislead you.

## What the device sends

A small subset of what `keylab-mk3-sysex.md` documents. All packets share the
envelope `F0 00 20 6B 7F 42 <payload> F7`, and every payload here is in the
`00 02 04` DAW screen family.

```
connect     00 02 05 01
button      00 02 04 <id> | 00 <02 off / 03 on> 00 | 03 00 00
                          | 04 <r> <g> <b> 00 | 05 <text> 00
list title  00 02 04 28 | 00 <title> 00 | 02 <hi> <lo> 00
list item   00 02 04 15 | 00 <text> 00 | 02 00 00 | 03 <hi> <lo> 00
cursor      00 02 04 15 | 03 <hi> <lo> 00
```

The two bytes in the title packet are the number of items you actually push in,
high byte first — not the length of some longer list you are showing a window
of. Send the wrong number and the firmware places the cursor in the wrong
frame.

Three things that are easy to get wrong and hard to diagnose:

- **Send each packet as one complete message**, not as a stream of single
  bytes for `[midiout]` to reassemble. Byte by byte loses 2-4% of packets,
  which shows up as "every thirtieth press the screen doesn't update".
- **Space screen packets out.** Two `00 02 04` packets sent back to back can
  race on the device and one is dropped. This POC queues them 60 ms apart.
  Whole bursts are worse: a few dozen SysEx packets at once can freeze the
  keyboard until it is power-cycled.
- **Plain ASCII only**, 32-126. Anything else costs you the character or the
  whole packet.

The cursor number is not the line on screen. The firmware derives the line from
it, anchored on the fourth line and framed by the list length sent in the title
packet. When the whole list fits on screen the number is simply the index —
without that special case the arithmetic goes negative and the box moves
backwards on short lists.

Playback uses `cue_points <n> call jump`, not `current_song_time`. That
property cannot be set: it reads back unchanged whatever the transport is
doing.
