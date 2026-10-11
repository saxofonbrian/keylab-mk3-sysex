# Arturia KeyLab mk3 — screen SysEx, LEDs and CC

Reverse-engineered notes for the KeyLab 61/88 mk3. Applies to both sizes; only
the MIDI port name differs. This is **not** official documentation and is not
endorsed by Arturia.

Mapped 27-28 August 2026, extended 4-5 September, reviewed against working
production code 6 September 2026, list item fields and the popup-over-list MIDI
pauses added 11 October 2026. Sources: Arturia's own `config.lua`
(their MainStage device script), a MIDI log captured from Analog Lab, direct
measurement on hardware, and two Max for Live JavaScript devices that drive a
live rig built on this protocol.

Confidence markers used throughout:

- **[measured]** — verified on hardware.
- **[code]** — the packet or CC is used in production code and works in daily
  use. Stronger than a guess, weaker than a measurement: it has never been
  isolated and varied, so it is unknown which fields actually carry the effect.
- **[unverified]** — read out of code or a log, never tested.
- **[guess]** — derived from a pattern, never observed.
- **[rejected]** — tried and found wrong. Listed so nobody spends an
  afternoon on it twice.

---

## Envelope

    F0 00 20 6B 7F 42 <payload> F7

## Two screen families

Byte 7 of the payload decides which screen world you are in:

| Family | Used by | Layout |
|---|---|---|
| `00 02 04` | The MainStage script, DAW mode | 7-line list (one column, or with icon panel), footer button rows |
| `00 01 04` | Analog Lab, Arturia mode | 7-line list (one column, or with icon panel) |

**[measured]** The list layout is chosen by the title packet's control byte, not
by the family: `14` is one column across the whole width, `28` is a list with an
icon panel on the right. Both work in both families. The two-column list with a
divider (`0x31`) exists only in `00 02 04`.

**[measured]** The families are isolated. Packets from one are ignored while
the other's page is displayed, in either order.

`00 02 03` is a third family: pure LED packets, no screen effect. See the LED
section.

## Mode and handshake

    00 02 05 01   DAW connect        (CONNEXION_DAW in config.lua)
    00 02 05 00   DAW disconnect
    00 01 05 01   Arturia connect
    00 01 05 00   Arturia disconnect

**[measured]** Connect only works when the unit is already in the corresponding
mode. It is a handshake, not a mode switch.

**[code]** Sending DAW connect on every init is cheap and makes a rig
independent of what else has touched the port. Without it the screen stayed
dead whenever the keyboard was powered on after the DAW.

**[measured]** There is no known SysEx that changes mode deliberately. The
switch is physical: the Prog key plus screen button 5 (Arturia) or 6 (DAW).
`21 11 40 02 00 <n>` from the KeyLab **Essential** mk3 guide does nothing on
this model.

**[measured] 5 Sep 2026 — the exception:** a write in the configuration family
`21 03 40` throws the unit into **User mode** immediately. It is not a usable
mode switch: there is no way back to the DAW bank without a physical press, so
in practice it is a side effect to avoid rather than a feature. See the pad
bank section.

---

# CC in DAW mode (DAW port, channel 1)

Everything below is momentary unless stated: 127 on press, 0 on release.

## Screen buttons — CC45-52

**[measured] 23 Jul 2026** with a MIDI monitor, all eight individually. They
form one unbroken CC range, top row first:

| Button | Row | SysEx id | CC |
|---|---|---|---|
| 1 | bottom | `0x01` | 49 |
| 2 | bottom | `0x02` | 50 |
| 3 | bottom | `0x03` | 51 |
| 4 | bottom | `0x04` | 52 |
| 5 | top | `0x05` | 45 |
| 6 | top | `0x06` | 46 |
| 7 | top | `0x07` | 47 |
| 8 | top | `0x08` | 48 |

Note that the SysEx id follows the button's own number while the CC range runs
the **other way**: the top row has the lower CC numbers. Channel 1, not channel
10 as the pads use.

**[measured]** The screen buttons have RGB LEDs, but they have **no address in
the `00 02 03` family**. A probe of ids 0-127 there drew responses only from
the transport/function row and the twelve pads. The buttons' light is set
purely by the colour field (field 04) of the footer packet in `00 02 04` — the
same packet that writes the label. That is why `00 02 04 <id> | 00 02 00 |
04 <rgb> 00 | 05 00` with an empty text field lights a button without writing
anything on it.

Look for these LEDs in the LED family and you find nothing, and it is easy to
conclude the buttons have no light. They do.

Arturia's `config.lua` numbers them differently: it calls them `F_Button1-8`,
where F_Button1-4 map to SysEx ids 5-8 and F_Button5-8 map to ids 1-4. Their
"F_Button1" is the button called 5 here.

SysEx id 0x08 is **[code]**, not measured in isolation — only 0x01-0x04 have
been tested individually. It draws correctly in use.

## Transport / utility row

**[measured]** Physical order, left to right: Tap, Rec, Play, Stop, Cycle,
Back, Forward, Metro, Redo, Undo, Quantize, Save.

| Button | CC | Source |
|---|---|---|
| Stop | 20 | [measured] |
| Play | 21 | [measured] |
| Rec | 22 | [measured] |
| Tap | 23 | [measured] |
| Cycle | 24 | [measured] |
| Back | 25 | [measured] |
| Forward | 26 | [measured] |
| Metro | 27 | [measured] |
| Save | 41 | [measured] |
| Quantize | 42 | [measured] |
| Undo | 43 | [measured] |
| Redo | 44 | [measured] |

**The row splits into two groups, and the second one runs backwards.** The
first eight buttons ascend across 20-27. The last four sit at 41-44 and
**descend** relative to their physical order: Redo (position 9) = 44, Undo =
43, Quantize = 42, Save (position 12) = 41. The row is neither contiguous nor
consistently directed — read the table, don't calculate.

Note also that 41-44 butt directly up against the screen buttons' 45-52. The
twelve centre buttons share a number range with the screen buttons rather than
with each other.

## Jog wheel

**[code]**

| Function | CC | Behaviour |
|---|---|---|
| Turn | 116 | centred on 64: >64 clockwise, <64 counter-clockwise |
| Click | 117 | momentary, 127/0 |

The wheel is not a relative +1/-1 encoder: the value moves further from 64 the
faster you turn, saturating at 61 and 67. Using only the sign and treating each
message as one step keeps a list from running away on a fast turn.

**[measured] 5 Sep 2026** The wheel click is genuinely momentary and reports
hold: 127 on press, 0 on release 938 ms later, no bounce. It can therefore
carry both short and long press gestures.

## Faders and encoders — movement

**[measured]** Every control has **two** CCs: one for touch and one for
movement. They sit far apart and must be kept separate.

| No. | Fader | Encoder |
|---|---|---|
| 1 | 105 | 91 |
| 2 | 106 | 92 |
| 3 | 107 | 94 |
| 4 | 108 | 95 |
| 5 | 109 | 96 |
| 6 | 110 | 97 |
| 7 | 111 | 102 |
| 8 | 112 | 103 |
| 9 | 113 | 104 |

The faders are contiguous, 105-113, and send absolute 0-127. The encoders are
not: 93 and 98-101 are not part of the range. Read the table, don't
calculate.

**[measured]** In DAW mode the encoders send **relative** values, as 64 ± delta
(two's complement around 64) — not absolute 0-127 as they do in Arturia mode.
Same encoding as the jog wheel. This means an encoder's actual value does not
exist in the hardware: it has to be accumulated in the receiver, and there is
nothing to read it back from at startup.

**[measured]** Faders 1-4 (CC105-108) can be swallowed by the DAW itself if
they happen to be MIDI-mapped there, so they never reach a device. It looks
like a dead fader and is not.

## Faders and encoders — touch

**[measured] 24 Aug 2026:** every fader and encoder has its own touch CC, 127
on contact and 0 on release. The ranges are **not** contiguous:

| Control | Touch CC |
|---|---|
| Fader 1-4 | 28, 29, 30, 31 |
| Fader 5-9 | 33, 34, 35, 36, 37 (CC32 is not used) |
| Encoder 1-8 | 8-15 |
| Encoder 9 | 38 |

So the natural assumption "CC28 plus fader index all the way up" is wrong.

**[measured] — important, and not a software bug:** the KeyLab's own on-screen
readout for faders and encoders is triggered by **capacitive skin contact**,
not by the value change. Put a sheet of paper between finger and fader and you
get no touch CC and no screen readout at all, even though the value CC is sent
and the sound follows. Nudge a fader with a fingernail and the display stays
silent. Encoders cannot physically be turned without skin contact, which is why
they always appear to work. This explains a long list of "dropped packets" that
were never dropped packets.

## Pads in the DAW bank

**[code]** The pads send Notes **0-11** for pads 1-12 on the DAW port, Note On
on press and Note Off on release. The other banks send entirely different notes
out of the physical MIDI port — see the pad bank section.

---

# Family `00 02 04` — DAW / MainStage

## Footer buttons

    00 02 04 <id> <fields...>

Field ids and meaning (all optional — omitted when `nil`):

| Field | Content |
|---|---|
| `00` | type, 0-15 |
| `01` | line_number |
| `02` | color_type |
| `03` | icon |
| `04` | RGB, three bytes |
| `05` `06` `07` | text line 1, 2, 3 |

### Arturia's own button format — use this one

**[measured] 12 Aug 2026** by recording Arturia's own Ableton control script
with a MIDI monitor. Their packet for button id 0x01, off and on:

    00 02 04 01 | 00 02 00 | 03 00 00 | 04 7F 7F 7F 00 | 05 "2" 00
    00 02 04 01 | 00 03 00 | 03 00 00 | 04 7F 58 00 00 | 05 "2" 00

So: **field 00 = 0x03 on / 0x02 off**, plus a constant `03 00 00`. Arturia does
not send fields 01 and 02 at all.

An earlier home-grown packet sent field 00 = always 04, omitted field 03 and
included fields 01/02. It worked roughly 29 times out of 30. After switching to
Arturia's structure the display got visibly crisper and the every-30th-press
failure disappeared. The firmware accepts extra or missing fields most of the
time, but the deviation makes parsing fragile.

**[measured]** The field values are **context-dependent per button id**, not
universal. For button id 0x02 Arturia's script uses `00 0B` / `00 0A` and a
constant `03 0D` — neither the same on/off pair nor the same field 03 as id
0x01. Any new id needs measuring individually.

### Measured button results (id 8)

- **[measured]** A colour field alone, without the type field, is ignored.
- **[measured]** A bare id with no fields does nothing.
- **[measured]** `00 02 00 | 04 <rgb> 00 | 05 00` → **LED lights, no text**.
  The explicitly empty text field is what does it; without field 05 a box is
  drawn.
- **[measured]** `00 00 00 | 04 <black> 00 | 05 00` → **clears the slot**. This
  is the reset command. **[code]** Sending it for ids 1-8 in turn works as a
  "clear the screen" routine.
- **[measured]** Type 00 occupies the slot without taking colour.
- **[measured]** `line_number` 0-7 does not change row height.

### Space cost

- **[measured]** Top button row empty → the list has 5 lines.
- **[measured]** Top button row active → 3 lines.
- **[measured]** The cost is **per row, not per button**. Light one of 5-8 and
  the other three are free.
- **[measured]** The row can be cleared again during play, so the change is not
  locked in.

This applies **only** to the 0x31 list. In the 0x14/0x28 + 0x15 lists the buttons cost
nothing — see below.

## Popup (0x1E)

    00 02 04 1E | 00 <value text> 00 | 01 <name> 00 | 02 <0-127> 00 | 04 <rgb> 00

**[code]** The general-purpose acknowledgement popup. Field 02 is a hardware
value 0-127 (used as 0 = off, 127 = on when there is no real value).

**[measured]** An extra `03 01 00` was sent until 12 Aug 2026. Arturia does not
send it, and surplus fields make parsing fragile. It has been removed.

**[unverified]** Whether the device actually colours the popup layout from
field 04 has never been confirmed — the field is sent, the effect is unseen.

## Fader / encoder readout (0x1D and 0x1C)

Arturia's fader packet, measured with a MIDI monitor:

    00 02 04 1D | 00 "-6.912 dB" 00 | 01 "Volume" 00 | 02 55 00 | 04 00 6C 7A 00

**[measured]** Same lesson as the buttons: an additional `03 01 00` was being
sent and has been removed.

**[measured] 24 Aug 2026 — tried and rejected:** using the encoder layout
(0x1C) instead of 0x1D for faders made things worse — lag and more missing
updates.

Check the skin-contact finding above before debugging "missing fader updates".
That is hardware behaviour, not protocol.

### MIDI pauses when a readout is drawn over a list

**[measured] 11 Oct 2026.** With a list on screen, every changing 0x1D readout
makes the keyboard stop sending MIDI for about 60 ms out of every ~100 ms. It
is the keyboard's own output that pauses, so it shows in a MIDI monitor on the
way in, not just in the host. Played notes at a steady tempo while a fader is
moved arrive in bunches (note-on spacing alternates ~100 / ~190 ms instead of a
steady ~140 ms), and the fader's own CC values skip 2-3 steps at a time instead
of 1. An empty list (title only) pauses too, for less time (~43 ms in ~80 ms).

Without a list on screen there are no pauses — readouts alone, readouts with
footer buttons, and readouts with LEDs all stayed clean. Arturia's own script
sends ~44 readouts a second without trouble because it never has a list up.

**[measured]** None of these changes the pauses: the send rate (the same at 30
and at 70 readouts per second); the readout's fields `03` and `04`, including
the values Analog Lab sends (`03 01`, `04 01 5D 7C`) and Arturia's (`04 00 6C
7A`); echoing the touch CC back; writing the Live parameter; resending DAW
connect on touch.

**[measured] The fix is in the list, not the readout.** The pauses are gone when
the list is drawn the way Analog Lab draws its own:

- title `0x28` **with the icon field `01` on every item**, or
- title `0x14` (the one-column list, see below) with items that have no field
  `01`.

The combination that pauses is title `0x28` with items **without** field `01`.
Field `03` on the items (the
line index, or `00 00`) makes no difference to the pauses.

**[measured]** Drawing the readout or the list in family `00 01 04` while the
keyboard is in DAW mode removes the pauses only because nothing is drawn at all.
The families are isolated, see above.

## List — 7 lines (0x14 or 0x28, plus 0x15)

**[measured]** This exists in this family too, even though `config.lua` does not
know about it. It is the same model Analog Lab uses in `00 01 04`, and it is
the better of the two list widgets.

    00 02 04 28   title and the list's full length. Sent ONCE. Two columns:
                  the right-hand side is an icon panel, so every item needs
                  the icon field 01 (see the pauses above)
    00 02 04 14   the same title packet, but one column across the whole
                  width, no icon panel — Analog Lab's GENRES list
    00 02 04 15   push an item in at the BOTTOM — the view scrolls down
    00 02 04 16   push an item in at the TOP — the view scrolls up
    00 02 04 33   same as 0x16

The packets as actually sent **[code]**:

    title:   00 02 04 14 | 00 <title> 00 | 02 00 <length> 00
    item:    00 02 04 15 | 00 <text> 00 | 02 00 00 | 03 00 <index> 00
    cursor:  00 02 04 15 | 03 00 <index> 00

with `28` instead of `14` in the title and `01 <icon id>` in every item, the
item packet is:

    item:    00 02 04 15 | 00 <text> 00 | 01 <icon id> | 02 00 00 | 03 00 <index> 00

Field 02 in the title packet is two bytes, **high byte first**: `02 <hi> <lo>
00`. The same goes for the items' field 03. Everything is masked to 7 bits, so
lists longer than 127 items would need the high byte — never attempted.

### Item fields

**[measured] 11 Oct 2026**, in the two-column list (title `0x28`):

| Field | Content |
|---|---|
| `00` | text |
| `01` | icon id, one byte. Shown in the icon panel for the line the cursor is on, and it changes as the cursor moves |
| `02` | two bytes. The **first** byte is the text colour: `00` white, anything else blue. The second byte has no visible effect |
| `03` | two bytes, the line index. Analog Lab sends `00 00` in its icon list, and `00 00` or the index in its one-column list; either is fine here |

Icon ids, as seen on screen with the cursor on each line: `00` the Arturia logo,
`01` a string instrument, `02` a trumpet, `03` a drum, `04` keys, `05` a
keyboard. That fits Analog Lab's categories (Bass 1, Electric Piano 4, Keys 5,
Lead 6, Organ 7, Pad 8, Piano 9, see the Analog Lab section). Ids above 5 were
not looked at on this family's screen.

Text colour: values 1-6, 8-112 (every 4th) and 127 were tried and all gave the
same blue. There is no colour scale here, just white or blue.

**[measured] 31 Aug 2026:** field 02 was swept 0-20 to see whether the cursor box
colour could be controlled. The box did not change. See above: the field
colours the item's own text.

### How the list behaves

- **[measured]** A title packet (`0x28`, or `0x14`) must run first. Without it,
  0x15 and 0x16 are ignored.
- **[measured]** The ceiling is 7 visible lines.
- **[measured]** **LEDs in the top button row cost nothing here.** This is the
  decisive difference from the 0x31 list. All eight buttons can carry both
  colour and text while the list stays at 7 lines.
- **[measured]** The view always shows the last seven items pushed in. The
  firmware does **not** scroll on its own — it only moves the cursor box. To
  scroll, push an item in at the end you are moving toward.
- **[measured]** Re-sending an item with 0x15 appends it at the bottom as a new
  entry regardless of its index field. The index does not position the item.
- **[measured]** Items' field 03 moves the cursor as they are pushed in, so the
  box ends up on the last item sent. Set the cursor last.
- **[measured]** A title sent alone, with no items after it, resets the list.
- **[code]** There is no "close list" command. A list is replaced by sending a
  new title with new items; to clear the screen entirely, use the button reset
  packet.

### The cursor number

**[measured]** The number in the cursor packet is not the line on screen. The
firmware works out the box's line from that number, anchored on line 4 and
framed by the list's full length (field 02 of 0x28). Measured with a 14-item
list:

| Number | Line |
|---|---|
| 0-3 | 1-4 |
| 4-10 | 4 (unchanged) |
| 11-13 | 5-7 |

To place the box on a specific line, compute the number backwards. The rule as
implemented, with the anchor on the 4th line (index 3) and 7 visible lines:

    whole list visible (length <= 7):   number = cursor index
    otherwise, line = index - top line:
        line <= 3:   number = line
        line >  3:   number = line + (length - 7)

**[measured] 3 Sep 2026:** the short-list special case is not cosmetic. Without
it `length - 7` goes negative and the cursor subtracts instead of adding: with
6 lines the box froze on the 5th item, with 5 lines it moved **backwards**.

**[rejected]** `0x15` with field 04 looks like a cursor command in one probe,
but does not move the box. Field 03 is the right one.

**Side effect:** 0x16 switched off a button LED during the sweep. Not
re-tested in normal use.

## List — two columns (0x31)

    00 02 04 31   refresh / init
    00 02 04 32   append          field 00 = left, field 04 = right
    00 02 04 33   prepend
    00 02 04 34   list heading
    00 02 04 01   header zone

0x31 fields: `00` preset, `02` left_size, `06` right_size, `0A`/`0B` column
labels. Arturia's script has fields `03`/`07` (global position) and `05`/`09`
(select) commented out.

Note that `01` outside the 0x31 widget is footer button 1 and nothing else —
the "header zone" reading applies only inside the widget.

- **[measured]** The size fields only control which end is shaded — not the
  number of visible lines.
- **[measured]** The number of appended items does not change the window either.
- **[measured]** Omit `0A`/`0B` and the "SET"/"PRESET" labels disappear, but the
  band and divider remain. They belong to the widget, not to the fields.
- **[unverified]** `SCREEN_ROWS = 5` is hard-coded in Arturia's script, and they
  never put content in the top button row while a list is displayed.
- **[measured] 26 Aug 2026:** the working chain in `config.lua` is
  `SetCenterList()` → `Screen_List_Refresh` (0x31) / `Screen_List_Header`
  (0x34), `AppendToList()` → 0x32, `PrependToList()` → 0x33, `SetListCursor()`
  → 0x32 with field ids 03/07 instead of 00/04. `Screen_List_Add_Preset` is
  dead code with undefined variables and is called from nowhere — do not read
  it as documentation.

---

# Family `00 02 03` — LEDs

    00 02 03 <id> <r> <g> <b> 00

One packet per LED, RGB 0-127 per channel. No screen effect.

## Transport row LED ids

**[measured]** Every id measured so far fits the formula **LED id = physical
position + 3** (position 1-indexed from Tap) without exception:

| Position | Button | LED id | Source |
|---|---|---|---|
| 1 | Tap | 4 | [measured] |
| 2 | Rec | 5 | [guess] |
| 3 | Play | 6 | [measured] |
| 4 | Stop | 7 | [measured] |
| 5 | Cycle | 8 | [measured] |
| 6 | Back | 9 | [measured] |
| 7 | Forward | 10 | [measured] |
| 8 | Metro | 11 | [measured] |
| 9 | Redo | 12 | [guess] |
| 10 | Undo | 13 | [guess] |
| 11 | Quantize | 14 | [guess] |
| 12 | Save | 15 | [guess] |

Tap and Metro were confirmed with an LED sweep (ids 0-20, a white blink per id)
on 4 Sep 2026. They sat furthest from the originally known ids, so the formula
is on solid ground — but it remains a guess for the four remaining buttons.

**[rejected]** "LED id = position in the order the CCs were measured" — this
collapsed as soon as Tap and Rec were checked against Play and Stop's known
ids. The formula uses the PHYSICAL button order.

## Pad LED ids

**[measured]** `LED id = 82 + pad number - 1`, i.e. 82-93 for pads 1-12.

The evidence per id is uneven, which is worth knowing before building on it:

| Pad | LED id | Source |
|---|---|---|
| 1 | 82 | [guess] — from the formula |
| 2 | 83 | [code] |
| 3 | 84 | [measured] |
| 4 | 85 | [measured] |
| 5-8 | 86-89 | [code] |
| 9-10 | 90-91 | [guess] — from the formula |
| 11-12 | 92-93 | [measured] |

Pads 9-10 look like an odd gap between measured neighbours. They were part of
a sweep whose session ended with the device frozen (see send limits below), so
those results were discarded rather than trusted. It is cheap to settle: give
each pad its own colour in a normal session and look.

## Send limits

**[measured] WARNING:** 36 LED packets sent in one burst from SysEx Librarian
**freezes the device**, requiring a power cycle. When testing, send in small
groups; four at a time is plenty.

**[code]** Sending 12 packets at once (repainting all pads) is fine in practice.
The limit is somewhere between 12 and 36.

**[measured]** These packets only work **in the DAW bank**. In bank D nothing
happens — the bank paints its own pads from the stored User program.

---

# Family `00 01 04` — Arturia / Analog Lab

**[measured]** Works without Analog Lab running.

## Buttons

    00 01 04 <id> 00 <type> 00 | 01 01 00 | 02 <sel> 00 | [04 <rgb> 00] | 05 <text> 00 | 06 00

Ids 1-4 are the tabs along the bottom, id 8 is the label at the top right.
Type: `00` clears, `02` top label, `04` and `06` tabs.

## List — 7 lines, with icon panel (0x28)

    00 01 04 28   page title; field 02 = total count (for the scrollbar)
    00 01 04 15   add item

Item fields:

| Field | Content |
|---|---|
| `00` | text |
| `01` | icon id |
| `02` | text colour — 0 white, 1 blue |
| `03` | two bytes, unknown |

Icon ids from the log: Bass 1, Electric Piano 4, Keys 5, Lead 6, Organ 7,
Pad 8, Piano 9. **[unverified]** From Analog Lab's INSTRUMENTS list (log, 11 Oct
2026): Acid 0x11, ARP 2600 0x10, Aug. STRINGS 0x15, Aug. VOICES 0x16, B-3 0x18,
Buchla Easel 0x19, Clavinet 0x1E.

The same icon and colour fields work in the DAW family's list; see the item
fields table there. In `00 02 04` the text colour is the first of the field's
two bytes and any non-zero value gives blue.

### The one-column list (0x14)

**[unverified]** From a log of Analog Lab switching to its GENRES list, with the
screen showing one column across the whole width and no icon panel:

    title:   00 01 04 14 | 00 GENRES 00 | 02 00 <count> 00
    item:    00 01 04 15 | 00 <text> 00 | 02 00 00 | 03 00 <index> 00
    cursor:  00 01 04 15 | 03 00 <index> 00

The title control byte is `14` instead of `28` and the items have no field
`01`. The items sent when the list opens have `03 00 00`; the ones appended
while scrolling carry the line's index in field `03`. Scrolling sends one item
per step (append at the bottom) followed by the cursor, which is the same
sequence the `00 02 04` list uses. The same title packet works in family `00 02 04`, see
that section.

- **[measured]** There is no refresh packet. A title packet with the count resets the list.
- **[measured]** Field `04` overwrites the text **and** changes the icon. Field
  `05` also changes the icon. Neither produces a second text column.
- **[measured]** Fields `06` and `07` produce only a heading, no item.
- **[measured]** The right-hand side is an icon panel. There is no second text
  column in this layout.

## Control bytes in this family

From a sweep of 0x00-0x3F:

| Byte | Effect **[measured]** |
|---|---|
| `05` `06` `07` | lights LED 5, 6, 7 |
| `0B` `0D` | writes text in the middle of the screen |
| `0C` `0E` `26` `2E` | clears the screen |
| `15` | add list item (see above) |
| `16` | as 0x15, but white text |
| `17` `18` | small heading plus grey popup |
| `19` | two-line popup |
| `1A` | popup, right-aligned text |
| `1B` `1E` | popup, several lines in sequence |
| `1F` | clears the heading, writes in small type |
| `22` | popup with a half-filled bar and arrows |
| `23` | empty bar with arrows |
| `25` | a kind of welcome screen |
| `27` | colour palette at top left — **unexplored** |
| `28` | page title |
| `29` | clears the page, leaves a piano icon |
| `2A` | large piano icon |
| `2F` | keyboard with colour marking, looks like key range — **unexplored** |
| `31`-`34` | the MainStage list widget, here too |

- **[measured]** 0x32, 0x33 and 0x34 only work if 0x31 has run first.
- **[measured]** 0x31 drags the MainStage divider and footer band along with it,
  even inside this family.
- **[measured]** The two list widgets (0x15 and 0x31) are mutually exclusive.
  Whichever was drawn last takes the screen.

## Other control bytes in the DAW family

From a sweep of 0x00-0x3F **[measured]**:

| Byte | Effect |
|---|---|
| `02` `04` `08` | lights LED 2, 4, 8 |
| `14` | heading; also the title of the one-column list, see the list section |
| `15` | list item (requires a title, 0x14 or 0x28, first) |
| `16` | as 0x15, but switches off an LED and pushes content up |
| `19` | popup |
| `1C` | encoder readout |
| `1D` | fader readout |
| `1E` | pad / general popup |
| `25` | welcome screen |
| `27` | colour picker |
| `28` | page title, initialises the list |
| `29` | grey keyboard icon at the top with a heading |
| `2A` | grey keyboard icon in the middle |
| `2F` | keyboard drawing for key range |

---

# CC in Arturia mode

A completely different mapping applies here. Mixing the two up looks like a
firmware fault.

## Screen buttons

**[measured]** Channel 1, momentary:

| Button | CC |
|---|---|
| 1 | 60 |
| 2 | 61 |
| 3 | 62 |
| 4 | 63 |
| 5 | 56 |
| 6 | 57 |
| 7 | 58 |
| 8 | 59 |

The two groups are not contiguous: 1-4 are 60-63, 5-8 are 56-59.

## Faders and encoders

**[measured]** Channel 1. The order is irregular — read the table, don't guess.
The encoders send absolute values 0-127, not relative steps.

| No. | Fader | Encoder |
|---|---|---|
| 1 | 73 | 74 |
| 2 | 75 | 71 |
| 3 | 79 | 76 |
| 4 | 72 | 77 |
| 5 | 80 | 93 |
| 6 | 81 | 18 |
| 7 | 82 | 19 |
| 8 | 83 | 16 |
| 9 | 85 | 17 |

The faders use 72, 73, 75, 79-83 and 85. The gaps — 74, 76, 77, 78 and 84 — are
not random: 74, 76 and 77 are encoders 1, 3 and 4.

---

## Pad banks and User mode configuration

The other pad banks are **layers over the same 12 physical pads**. There is no
pad 13: pad numbers are 1-12 in every bank, and the LED ids are the same.

### The banks' notes

**[measured]** Bank notes come out of the **physical MIDI port** on channel 10,
not the DAW port. Bank A starts at note 36, B at 48, C at 60, D at 72.
(B and C are inferred from the pattern, not measured.)

MCC's own numbering is chromatic: Pad 1 = 36, Pad 2 = 37, ... Pad 12 = 47 in
bank A. A DAW that shows a drum-rack layout (C at bottom left) displays the
rows in the opposite order, so the pad at bottom left is MCC's Pad 1.

**[measured]** Bank D's order as the pads sit physically:

    pad  9 10 11 12  =  note 72 73 74 75
    pad  5  6  7  8  =  note 76 77 78 79
    pad  1  2  3  4  =  note 80 81 82 83

That is, C4 at bottom left up to B4 at top right.

### The configuration family `21 03 40` — where the colours live

Mapped 5 Sep 2026 by diffing two MCC saves that differed only in bank D pad 1's
colour (yellow vs white). Exactly **one byte** differed.

    F0 00 20 6B 7F 42 21 03 40 <lo> <hi> <value> F7   write one byte
    F0 00 20 6B 7F 42 20 09 40 <lo> <hi> F7           read one byte
    F0 00 20 6B 7F 42 21 09 40 <lo> <hi> <value> F7   the device's reply
    F0 00 20 6B 7F 42 06 03 F7                        commit / save
    F0 00 20 6B 7F 42 24 03 20 <idx> <page> <text> F7 names ("Pad 4", "Fader 1")

The address is `lo + hi * 128`.

**Pad record, 12 bytes, stride 0x0C:**

    <colour>, 01, <note+8>, <note>, <pad number>, 0A, 00, 00, 7F, 00, 7F, 00

Each pad has four records, one per bank. Pad 1 sits at 0x00D3 (bank A, note
36), 0x00DF (B, 48), 0x00EB (C, 60), 0x00F7 (D, 72).

**[measured]** The colour is **a single palette index**, not an RGB triple as in
the `00 02 03` family. Known values: `0x28` = default/white, `0x05` = yellow.
The rest of the palette is unmapped — the easiest route is to set pad 1 to a
colour in MCC, save, and read the index out of the log.

**[measured]** A write takes effect immediately, even without a commit, and
survives a mode change. But it throws the unit into User mode, so it **cannot**
be used for dynamic colour feedback while playing.

### What bank D is good for, then

Colours are set once in MCC and stay put — static, not feedback. The MIDI on
channel 10 can be captured from the physical port, and the screen can still
show popups with names. You get feedback, just on the display rather than in
the pad.

---

## Sending, in practice

Five things that appear in no protocol table but decide whether any of this
works:

1. **One complete message per call.** **[measured] 20 Aug 2026:** sending a
   packet as a stream of raw single bytes for the receiver to reassemble loses
   2-4% of packets. Sent as one complete message: 111 presses without a single
   loss. That was the whole explanation for "every 30th or 40th press the
   screen doesn't update", and it explains why Arturia's own script never
   failed over the same cable and port.

2. **Space out screen packets.** **[code]** Two packets in family `00 02 04`
   sent back to back can race on the device itself and one gets dropped.
   Running all screen packets through a queue with 60 ms between them fixed it.
   LED packets (`00 02 03`) bypass the queue and have not shown the same
   problem.

3. **Don't send too many at once.** 12 packets at a time is fine, 36 freezes the
   device. See the LED section.

4. **Plain ASCII only.** **[code]** The screen only takes characters in the
   32-126 range. Accented and non-Latin characters must be transliterated, and
   typographic quotes and dashes replaced, or the character — or the packet —
   is lost.

5. **Text lengths.** Footer labels are cut at 12 characters and list lines at 18
   in the working rig. Those are chosen, tested values, not measured maxima.

6. **A list changes how the keyboard behaves under a fader readout.**
   **[measured] 11 Oct 2026:** with a title `0x28` list whose items lack field
   `01` on screen, every changing 0x1D readout makes the keyboard stop sending
   MIDI for ~60 ms in every ~100. Notes and fader CCs arrive late and in
   bunches. Use title `0x14`, or give every item an icon id. See the pauses
   section under the fader readout.

---

## Summary: choosing a layout

| | 0x14 + 0x15 | 0x28 + 0x15 | 0x31 list | 0x15 in the AL family |
|---|---|---|---|---|
| Lines | 7 | 7 | 5 | 7 |
| Columns | 1, full width | list plus icon panel | 2 | 1, with icon panel |
| Divider | no | no | yes | no |
| LEDs in the top row | free | free | cost 2 lines (5 → 3) | free |
| Family | `00 02 04` | `00 02 04` | both | `00 01 04` |
| Items need field `01` | no | yes | — | yes |

**0x14 + 0x15 in DAW mode is the right choice** for most purposes: 7 lines
across the full width, lights in both button rows, the DAW-mode CC mapping, and
no MIDI pauses when a fader readout is drawn over the list. Use 0x28 with icon
ids when you want the icon panel. Do not use 0x28 with items that have no
field `01`.

## Dead ends

- Colour without a type field in family 02 — ignored.
- `line_number` as a way to get a shorter footer row — no effect.
- Removing column labels to gain space — the band stays.
- Size and count fields as a way to get more lines — no effect.
- Family 02 appends on top of an Analog Lab page — ignored.
- `21 11 40 02 00 <n>` as a mode switch — no effect.
- `0x15` field 04 as a cursor command — it is field 03.
- `0x15` field 02 as cursor colour control — it colours the item's text (first
  byte: 0 white, anything else blue), not the cursor box.
- Field 02 of 0x28 set to the list's full length while only the visible items
  are sent — the firmware then computes the cursor in the wrong frame.
- `00 02 03` LED packets as a way to colour pads in bank D — ignored entirely;
  the bank paints itself from the User program.
- Id 94 and up as "pad 13" — there is no pad 13. The banks share the same 12
  LEDs.
- `00 01 03` as a presumed Arturia-mode counterpart to the pad LED family — not
  tested exhaustively, but overtaken by the discovery of the configuration
  family.
- "LED id = position in CC measurement order" — the correct formula uses the
  PHYSICAL button order.
- 0x1C (encoder layout) for the fader readout — worse than 0x1D.
- Spacing the fader readouts 20 ms apart, or removing the touch echo, to stop
  the MIDI pauses — the pauses are in the list, see the fader
  readout section.
- Drawing the readout, or the whole list, in family `00 01 04` while the
  keyboard is in DAW mode — nothing is displayed (no list, no readout, no
  scrolling).
- Debugging "missing fader updates on screen" — that is capacitive skin
  contact, not protocol.

## Unexplored

- Control bytes above 0x3F in both families.
- 0x27 (colour palette) and 0x2F (keyboard display).
- Lists longer than 127 items: would require the high byte in the length and
  index fields. Never attempted.
- Rec/Redo/Undo/Quantize/Save's LED ids — formula only, not measured.
- Pad LEDs 82, 90 and 91 — formula only, not measured.
- Palette indices in the `21 03 40` colour byte beyond 0x28 and 0x05.
- The rest of the configuration image: what the `21 09 40` region contains, and
  what the remaining bytes in the pad record (`0A`, the trailing `7F`) mean.
- Whether the read command `20 09 40` also covers the `03` region, so the whole
  configuration could be dumped without MCC.
- Whether the popup colour field (0x1E field 04) does anything at all.
- What the icon ids above 5 look like on the `00 02 04` screen, and what the
  second byte of the item colour field does.
- Why the icon field removes the MIDI pauses. Only the correlation is known.
- Bank B and C's notes are inferred from the pattern (48 and 60), not measured.

---

## Related work

The only comparable public material is a reverse-engineering guide for the
KeyLab **Essential** mk3 at
`github.com/PrzemekBarski/arturia-keylab-essential-mk3-programming-guide`.
It describes a different structure (screen at `04 01 60`, DAW connect
`02 0F 40 5A 01`) and does not transfer to this model.

Corrections and additions are welcome — particularly measurements for anything
marked [guess] or [code] above.
