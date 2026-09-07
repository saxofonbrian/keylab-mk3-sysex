# KeyLab mk3 — skærm-sysex, LED og CC

Gælder både 61 og 88; kun MIDI-portnavnet adskiller dem.

Kortlagt 27.-28. august 2026, udvidet 4.-5. september, gennemgået mod
produktionskoden 6. september 2026. Kilder: Arturias `config.lua`
(MainStage-script), MIDI-log fra Analog Lab, målinger direkte på hardware,
og de to Max for Live-filer der driver riggen (`keylab_arm.js`,
`keylab_list_arr.js`).

Markeringer:

- **[målt]** — efterprøvet på hardware.
- **[kode]** — pakken eller CC'en bruges i produktionskoden og virker i drift.
  Stærkere end et gæt, svagere end en måling: den er aldrig blevet isoleret
  og varieret, så det er uvist hvilke felter der bærer virkningen.
- **[ulæst]** — læst ud af kode eller log uden at være efterprøvet.
- **[gæt]** — udledt, ikke set.

---

## Indpakning

    F0 00 20 6B 7F 42 <payload> F7

## To skærmfamilier

Byte 7 i payload afgør hvilken skærmverden man er i:

| Familie | Bruges af | Layout |
|---|---|---|
| `00 02 04` | MainStage-scriptet, DAW-mode | Liste i to kolonner, footer-rækker |
| `00 01 04` | Analog Lab, Arturia-mode | Liste i én kolonne, ikonpanel til højre |

**[målt]** Familierne er isolerede. Pakker fra den ene bliver ignoreret mens den
andens side står, uanset rækkefølge.

`00 02 03` er en tredje familie: rene LED-pakker, uden skærm. Se LED-afsnittet.

## Mode og handshake

    00 02 05 01   DAW connect        (CONNEXION_DAW i config.lua)
    00 02 05 00   DAW disconnect
    00 01 05 01   Arturia connect
    00 01 05 00   Arturia disconnect

**[målt]** Connect virker kun når enheden i forvejen står i det tilsvarende mode.
Det er et handshake, ikke et mode-skift.

**[kode]** DAW connect sendes ved hvert Init i begge js-filer. Det er billigt og
gør riggen uafhængig af om AbleSet har været åbnet først — uden det stod
skærmen død når KeyLab'en blev tændt efter Ableton.

**[målt]** Der findes ingen kendt sysex der skifter mode *med vilje*. Skiftet er
fysisk: Prog-tasten plus skærmknap 5 (Arturia) eller 6 (DAW).
`21 11 40 02 00 <n>` fra KeyLab **Essential** mk3-guiden virker ikke på denne model.

**[målt] 5/9-2026 — undtagelsen:** en skrivning i konfigurations-familien
`21 03 40` river enheden over i **User mode** med det samme. Det er ikke et
brugbart mode-skift: der findes ingen vej tilbage til DAW-banken uden et fysisk
tryk, så det er i praksis en bivirkning man skal undgå, ikke en funktion.
Se afsnittet om pad-banker nedenfor.

---

# CC i DAW-mode (DAW-porten, kanal 1)

Alt herunder er momentant hvor intet andet står: 127 ved tryk, 0 ved slip.
Det er den mode riggen kører i; Arturia-modes helt anden mapning står
længere nede.

## Skærmknapper — CC45-52

**[målt] 23/7-2026** med MIDI Monitor, alle otte enkeltvis. De ligger i én
ubrudt række, øverste række først:

| Knap | Række | Sysex-id | CC |
|---|---|---|---|
| 1 | nederste | `0x01` | 49 |
| 2 | nederste | `0x02` | 50 |
| 3 | nederste | `0x03` | 51 |
| 4 | nederste | `0x04` | 52 |
| 5 | øverste | `0x05` | 45 |
| 6 | øverste | `0x06` | 46 |
| 7 | øverste | `0x07` | 47 |
| 8 | øverste | `0x08` | 48 |

Bemærk at sysex-id'et er knappens eget nummer, mens CC-rækken løber
**modsat**: øverste række har de laveste CC'er. Kanal 1, ikke kanal 10 som
pads'ene.

**[målt]** Skærmknapperne har RGB-LED, men de har **ingen adresse i
`00 02 03`-familien**. En probe af id 0-127 dér gav kun svar fra
transport-/funktionsrækken og de tolv pads. Knappernes lys sættes udelukkende
af farvefeltet (felt 04) i footer-pakken i `00 02 04` — altså samme pakke som
skriver etiketten. Det er derfor `00 02 04 <id> | 00 02 00 | 04 <rgb> 00 |
05 00` med tomt tekstfelt kan tænde en knap uden at skrive noget på den.

Leder man efter knappernes LED i LED-familien, finder man ingenting og
konkluderer let at de ikke har lys. Det gør de.

Det er en anden nummerering end `config.lua`'s: dér hedder de otte
`F_Button1-8`, hvor F_Button1-4 → sysex-id 5-8 og F_Button5-8 → sysex-id 1-4.
Arturias "F_Button1" er altså den knap der her hedder 5.

Sysex-id 0x08 er **[kode]**, ikke isoleret målt — kun 0x01-0x04 er
hardware-testet enkeltvis. I drift tegner den rigtigt.

## Transport/utility-rækken

**[målt]** Fysisk rækkefølge, venstre mod højre: Tap, Rec, Play, Stop, Cycle,
Back, Forward, Metro, Redo, Undo, Quantize, Save.

| Knap | CC | Kilde |
|---|---|---|
| Stop | 20 | [målt] |
| Play | 21 | [målt] |
| Rec | 22 | [målt] |
| Tap | 23 | [målt] |
| Cycle | 24 | [målt] |
| Back | 25 | [målt] |
| Forward | 26 | [målt] |
| Metro | 27 | [målt] |
| Save | 41 | [målt] |
| Quantize | 42 | [målt] |
| Undo | 43 | [målt] |
| Redo | 44 | [målt] |

**Rækken er delt i to grupper, og den anden løber baglæns.** De første otte
knapper ligger stigende på 20-27. De sidste fire ligger på 41-44 og **faldende**
i forhold til den fysiske rækkefølge: Redo (position 9) = 44, Undo = 43,
Quantize = 42, Save (position 12) = 41. Rækken er altså hverken sammenhængende
eller ensrettet — læs tabellen, regn ikke.

Bemærk også at 41-44 støder direkte op til skærmknappernes 45-52. De tolv
knapper i midten deler nummerrum med skærmknapperne, ikke med hinanden.

## Jog-hjul

**[kode]**

| Funktion | CC | Opførsel |
|---|---|---|
| Drej | 116 | centreret om 64: >64 med uret, <64 mod uret |
| Klik | 117 | momentant, 127/0 |

Hjulet er ikke et relativt +1/-1: værdien ligger længere fra 64 jo hurtigere
man drejer, og mætter ved 61 og 67. Riggen bruger kun fortegnet og lader hvert
bud være ét trin — ellers løber listen fra én ved raske drej.

**[målt] 5/9-2026** Hjul-klikket er ægte momentant og rapporterer hold: 127 ved
nedtryk, 0 ved slip 938 ms senere, ingen prel. Det kan altså bære både kort og
langt tryk.

## Fadere og encodere — bevægelse

**[målt]** Hver kontrol har **to** CC'er: en for berøring og en for bevægelse.
De ligger langt fra hinanden og skal holdes adskilt.

| Nr. | Fader | Encoder |
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

Faderne er sammenhængende 105-113 og sender absolut 0-127. Encoderne er ikke
sammenhængende: 93 og 98-101 indgår ikke. Læs tabellen, regn ikke.

**[målt]** Encoderne sender **relativt** i DAW-mode, som 64 ± delta (tos
komplement omkring 64) — ikke absolut 0-127 som i Arturia-mode. Samme kodning
som jog-hjulet. Det betyder at en encoders faktiske værdi ikke findes i
hardwaren; den skal akkumuleres i modtageren, og der er ingen kilde at læse den
tilbage fra ved opstart.

**[målt]** Fader 1-4's CC105-108 kan blive opsnappet af DAW'en selv hvis de er
MIDI-mappet dér — så når de aldrig frem til et device. Det ligner en død fader
og er det ikke.

## Fadere og encodere — berøring

**[målt] 24/8-2026:** hver fader og encoder har sin egen touch-CC, 127 ved
berøring, 0 ved slip. Serierne er **ikke** sammenhængende:

| Kontrol | Touch-CC |
|---|---|
| Fader 1-4 | 28, 29, 30, 31 |
| Fader 5-9 | 33, 34, 35, 36, 37 (CC32 bruges ikke) |
| Encoder 1-8 | 8-15 |
| Encoder 9 | 38 |

Den nærliggende antagelse "CC28 + faderindeks hele vejen op" er altså forkert.

**[målt] — vigtigt, og ikke en softwarefejl:** KeyLab'ens egen skærmvisning for
fadere og encodere udløses af **kapacitiv hudkontakt**, ikke af værdiændringen.
Lægges et stykke papir mellem finger og fader, kommer der ingen touch-CC og
ingen skærmvisning, selvom værdi-CC'en sendes og lyden følger med. Skub en
fader med neglen, og displayet tier. Encodere kan fysisk ikke drejes uden
hudkontakt og virker derfor altid. Det forklarer en lang række "pakketab" der
aldrig var pakketab.

## Pads i DAW-banken

**[kode]** Pads sender Note **0-11** for pad 1-12 på DAW-porten, med Note On ved
nedtryk og Note Off ved slip. (De øvrige banker sender helt andre noder ud ad
den fysiske port — se pad-bank-afsnittet.)

---

# Familie `00 02 04` — DAW / MainStage

## Knapper (footer)

    00 02 04 <id> <felter...>

Felt-id og betydning (alle valgfri — udelades hvis `nil`):

| Felt | Indhold |
|---|---|
| `00` | type, 0-15 |
| `01` | line_number |
| `02` | color_type |
| `03` | ikon |
| `04` | RGB, tre bytes |
| `05` `06` `07` | tekstlinje 1, 2, 3 |

### Arturias eget knap-format — brug dette

**[målt] 12/8-2026** ved at optage Arturias eget Ableton-kontrolscript med MIDI
Monitor. Deres pakke for knap-id 0x01, slukket og tændt:

    00 02 04 01 | 00 02 00 | 03 00 00 | 04 7F 7F 7F 00 | 05 "2" 00
    00 02 04 01 | 00 03 00 | 03 00 00 | 04 7F 58 00 00 | 05 "2" 00

Altså: **felt 00 = 0x03 tændt / 0x02 slukket**, plus et fast `03 00 00`.
Felt 01 og 02 sender Arturia ikke.

Vores tidligere pakke sendte felt 00 = altid 04, manglede felt 03 og sendte
felt 01/02. Den virkede ca. 29 gange ud af 30. Efter omlægningen til Arturias
struktur står displayet skarpere, og hver-30.-tryk-fejlen forsvandt.

**[målt]** Feltværdierne er **kontekstafhængige pr. knap-id**, ikke universelle.
Arturias eget script bruger for knap-id 0x02 værdierne `00 0B` / `00 0A` og et
konstant `03 0D` — hverken samme on/off-par eller samme felt 03 som id 0x01.
Skal et nyt id have samme behandling, skal det måles enkeltvis.

### Målte knap-resultater (id 8)

- **[målt]** Farvefelt alene (uden type) bliver ignoreret. Type skal med.
- **[målt]** Nøgen ID uden felter gør ingenting.
- **[målt]** `00 02 00 | 04 <rgb> 00 | 05 00` → **LED lyser, ingen tekst**.
  Det eksplicit tomme tekstfelt er det afgørende; uden felt 05 tegnes en boks.
- **[målt]** `00 00 00 | 04 <sort> 00 | 05 00` → **rydder pladsen**. Reset-kommandoen.
  **[kode]** Den kan sendes for id 1-8 i træk som en "ryd skærmen"-funktion.
- **[målt]** Type 00 optager pladsen uden at tage farve.
- **[målt]** `line_number` 0-7 ændrer ikke rækkehøjden.

### Pladsforbrug

- **[målt]** Øverste knap-række tom → listen har 5 linjer.
- **[målt]** Øverste knap-række aktiv → 3 linjer.
- **[målt]** Prisen er **per række, ikke per knap**. Tænder du én af 5-8, er de
  tre andre gratis.
- **[målt]** Rækken kan ryddes igen under spil, så skiftet er ikke låst.

Dette gælder **kun** 0x31-listen. I 0x28+0x15-listen koster knapperne ingenting —
se nedenfor.

## Popup (0x1E)

    00 02 04 1E | 00 <værditekst> 00 | 01 <navn> 00 | 02 <0-127> 00 | 04 <rgb> 00

**[kode]** Riggens universelle kvittering: "Slow"/"Fast", "Brake On", "Init",
mute-tilstande, drawbar-navne. Felt 02 er en hardware-værdi 0-127 (bruges som
0 = fra, 127 = til når der ikke er en rigtig værdi).

**[målt]** Et ekstra `03 01 00` blev sendt indtil 12/8-2026. Arturia sender det
ikke, og overskydende felter gør parsningen skrøbelig. Det er fjernet.

**[ulæst]** Om KeyLab'en rent faktisk farver popup-layoutet efter felt 04 er
aldrig verificeret — feltet sendes, men effekten er ikke set.

## Fader-/encodervisning (0x1D og 0x1C)

Arturias format for fader-visningen, målt med MIDI Monitor:

    00 02 04 1D | 00 "-6.912 dB" 00 | 01 "Volume" 00 | 02 55 00 | 04 00 6C 7A 00

**[målt]** Samme lære som ved knapperne: vi sendte derudover `03 01 00`, og det
er fjernet.

**[målt] 24/8-2026 — afprøvet og forkastet:** at bruge encoder-layoutet (0x1C) i
stedet for 0x1D til fadere gjorde det værre — læg og flere manglende
opdateringer. Foreslå ikke layout-skift igen.

Husk hudkontakt-fundet ovenfor før der fejlsøges på "manglende
fader-opdatering". Det er hardware, ikke protokol.

## Liste — 7 linjer i fuld bredde (0x28 + 0x15)

**[målt]** Findes også i denne familie, selvom `config.lua` ikke kender den.
Det er samme model som Analog Lab bruger i `00 01 04`. **Det er den listemodel
riggen bruger.**

    00 02 04 28   titel og listens fulde længde. Sendes ÉN gang.
    00 02 04 15   skub et punkt ind FORNEDEN — visningen ruller ned
    00 02 04 16   skub et punkt ind FOROVEN — visningen ruller op
    00 02 04 33   som 0x16

Pakkerne som de faktisk sendes **[kode]**:

    titel:    00 02 04 28 | 00 <titel> 00 | 02 00 <længde> 00
    punkt:    00 02 04 15 | 00 <tekst> 00 | 02 00 00 | 03 00 <nr> 00
    markør:   00 02 04 15 | 03 00 <nr> 00

Felt 02 i titelpakken er to bytes, **høj byte først**: `02 <hi> <lo> 00`.
Det samme gælder punkternes felt 03. Alt maskeres til 7 bit, så lister over
127 punkter kræver at hi-byten tages i brug — det er aldrig afprøvet.

**[målt] 31/8-2026:** punkternes felt 02 (to bytes, ukendt) blev sweepet 0-20
for at se om boksens farve kunne styres. Ingen effekt ved nogen værdi. Feltet
er ubrugt eller reserveret; send 0.

### Sådan opfører listen sig

- **[målt]** 0x28 skal køre først. Uden den bliver 0x15 og 0x16 ignoreret.
- **[målt]** Loftet er 7 synlige linjer.
- **[målt]** **LED i øverste knap-række koster ingenting her.** Det er den
  afgørende forskel til 0x31-listen. Alle otte knapper kan have både farve og
  tekst samtidig, og listen bliver på 7 linjer.
- **[målt]** Visningen viser altid de sidst indskubbede syv punkter. Firmwaren
  ruller **ikke** af sig selv — den flytter kun boksen. Vil man rulle, skubber
  man et punkt ind i den ende man bevæger sig imod.
- **[målt]** Gensendes et punkt med 0x15, tilføjes det som nyt nederst, uanset
  hvad der står i indeksfeltet. Indekset placerer ikke punktet.
- **[målt]** Punkternes felt 03 flytter markøren efterhånden som de skubbes ind,
  så boksen ender på det sidst sendte punkt. Sæt derfor markøren til sidst.
- **[målt]** 0x28 sendt alene, uden efterfølgende punkter, nulstiller listen.
- **[kode]** Der findes ingen "luk liste"-kommando. En liste erstattes ved at
  sende en ny 0x28 med nye punkter; skal skærmen ryddes helt, bruges
  reset-pakken på knapperne.

### Markørens tal

**[målt]** Tallet i cursor-pakken er ikke linjen på skærmen. Firmwaren regner
selv boksens linje ud fra tallet, med forankring på 4. linje og listens fulde
længde (felt 02 i 0x28) som ramme. Målt med en 14-sangs liste:

| Tal | Linje |
|---|---|
| 0-3 | 1-4 |
| 4-10 | 4 (uændret) |
| 11-13 | 5-7 |

Vil man have boksen på en bestemt linje, regnes tallet den anden vej. Reglen
som den står i koden, med `ANKER = 3` (0-indekseret 4. linje) og `VISIBLE = 7`:

    er hele listen synlig (længde <= 7):   tal = markørens indeks
    ellers, linje = indeks - øverste linje:
        linje <= 3:   tal = linje
        linje >  3:   tal = linje + (længde - 7)

**[målt] 3/9-2026:** særtilfældet for korte lister er ikke pynt. Uden det bliver
`længde - 7` negativ, og markøren trækker fra i stedet for at lægge til: med 6
linjer stod boksen stille på 5. punkt, med 5 gik den et skridt **tilbage**.
Repertoirelisten er altid længere end skærmen, så fejlen viste sig først da
presetlisterne kom til.

**[forkastet]** `0x15` med felt 04 ligner en cursor-kommando i én sondering,
men flytter ikke boksen. Felt 03 er den rigtige.

**Bivirkning:** 0x16 slukkede en knap-LED i sweepet. Ikke efterprøvet i drift.

## Liste — to kolonner (0x31)

    00 02 04 31   refresh / init
    00 02 04 32   append          felt 00 = venstre, felt 04 = højre
    00 02 04 33   prepend
    00 02 04 34   liste-overskrift
    00 02 04 01   header-zone

0x31-felter: `00` preset, `02` left_size, `06` right_size, `0A`/`0B` kolonne-etiketter.
Arturias script har udkommenteret felt `03`/`07` (global position) og `05`/`09` (select).

Bemærk at `01` uden for 0x31-widgeten er footer-knap 1 og intet andet —
"header-zone"-læsningen gælder kun inde i widgeten.

- **[målt]** Størrelsesfelterne styrer kun hvilken ende der tones — ikke antal
  synlige linjer.
- **[målt]** Antal tilføjede elementer ændrer heller ikke vinduet.
- **[målt]** Udelades `0A`/`0B`, forsvinder teksten "SET"/"PRESET", men båndet og
  skillelinjen bliver stående. De hører til widgeten, ikke til felterne.
- **[ulæst]** `SCREEN_ROWS = 5` er hårdkodet i Arturias script, og de sætter
  aldrig indhold i den øverste knap-række mens en liste står.
- **[målt] 26/8-2026:** den virkende kæde i `config.lua` er `SetCenterList()` →
  `Screen_List_Refresh` (0x31) / `Screen_List_Header` (0x34), `AppendToList()` →
  0x32, `PrependToList()` → 0x33, `SetListCursor()` → 0x32 med felt-id 03/07 i
  stedet for 00/04. `Screen_List_Add_Preset` er dødt kode med udefinerede
  variabler og kaldes ingen steder — læs den ikke som dokumentation.

---

# Familie `00 02 03` — LED'er

    00 02 03 <id> <r> <g> <b> 00

Én pakke pr. diode, RGB 0-127 pr. kanal. Ingen skærmvirkning.

## Transport-rækkens LED-id

**[målt]** Alle otte kendte id'er rammer formlen **LED-id = fysisk position + 3**
(position 1-indekseret fra Tap) uden undtagelse:

| Position | Knap | LED-id | Kilde |
|---|---|---|---|
| 1 | Tap | 4 | [målt] 4/9-2026 |
| 2 | Rec | 5 | [gæt] |
| 3 | Play | 6 | [målt] |
| 4 | Stop | 7 | [målt] |
| 5 | Cycle | 8 | [målt] |
| 6 | Back | 9 | [målt] |
| 7 | Forward | 10 | [målt] |
| 8 | Metro | 11 | [målt] 4/9-2026 |
| 9 | Redo | 12 | [gæt] |
| 10 | Undo | 13 | [gæt] |
| 11 | Quantize | 14 | [gæt] |
| 12 | Save | 15 | [gæt] |

Tap og Metro blev bekræftet med et LED-sweep (id 0-20, hvidt blink pr. id) den
4/9-2026. De lå længst fra de oprindeligt kendte id'er, så formlen står stærkt
— men er stadig et gæt for de fire resterende.

> Kommentarerne i `keylab_arm.js` kalder stadig Tap=4 og Metro=11 "et gæt".
> Den note er forældet: de blev bekræftet samme dag koden blev skrevet.

**[forkastet]** "LED-id = position i CC-målerækkefølgen" — brød sammen da
Tap/Rec blev prøvet mod Play/Stops kendte id'er. Formlen bruger den FYSISKE
knap-rækkefølge.

## Pad-LED-id

**[målt]** `LED-id = 82 + padnr - 1`, altså 82-93 for pad 1-12.

Kildegrundlaget pr. id er ujævnt, og det er værd at kende før man bygger på det:

| Pad | LED-id | Kilde |
|---|---|---|
| 1 | 82 | [gæt] — formel-udledt |
| 2 | 83 | [kode] |
| 3 | 84 | [målt] |
| 4 | 85 | [målt] |
| 5-8 | 86-89 | [kode] |
| 9-10 | 90-91 | [gæt] — formel-udledt |
| 11-12 | 92-93 | [målt] (Leslie/Brake lå der indtil 4/9-2026) |

En tidligere udgave af dette dokument påstod at 90-93 blev målt 5/9. Den
måling foregik i den session hvor enheden frøs (se advarslen nedenfor), og
sweep-resultaterne derfra er ugyldige. Sagen afgøres billigt: giv pad 9-12 hver
sin farve i en almindelig session og se efter.

## Sendegrænse

**[målt] ADVARSEL:** 36 LED-pakker sendt i én byge fra SysEx Librarian **fryser
enheden**, så den skal slukkes og tændes. Ved test: send i småbidder, fire ad
gangen er rigeligt.

**[kode]** Riggen sender 12 pakker på én gang (alle pads males om når en
instrumentknap holdes) uden problemer. Grænsen ligger altså et sted mellem 12
og 36.

**[målt]** Pakkerne virker **kun i DAW-banken**. Står hardwaren i bank D, sker
der ingenting — banken tegner selv sine pads ud fra det gemte User-program.

---

# Familie `00 01 04` — Arturia / Analog Lab

**[målt]** Virker uden at Analog Lab kører.

## Knapper

    00 01 04 <id> 00 <type> 00 | 01 01 00 | 02 <sel> 00 | [04 <rgb> 00] | 05 <tekst> 00 | 06 00

Id 1-4 er fanerne forneden, id 8 er etiketten øverst til højre.
Type: `00` rydder, `02` øverste etiket, `04` og `06` faner.

## Liste — 7 linjer, én kolonne

    00 01 04 28   sidetitel; felt 02 = samlet antal (til scrollbaren)
    00 01 04 15   tilføj punkt

Punkt-felter:

| Felt | Indhold |
|---|---|
| `00` | tekst |
| `01` | ikon-id |
| `02` | tekstfarve — 0 hvid, 1 blå |
| `03` | to bytes, ukendt |

Ikon-id fra loggen: Bass 1, Electric Piano 4, Keys 5, Lead 6, Organ 7, Pad 8, Piano 9.

- **[målt]** Ingen refresh-pakke. 0x28 med antallet nulstiller listen.
- **[målt]** Felt `04` overskriver teksten **og** skifter ikonet. Felt `05` skifter
  også ikonet. Ingen af dem giver en anden tekstkolonne.
- **[målt]** Felt `06` og `07` giver kun en overskrift, intet punkt.
- **[målt]** Højresiden er ikonpanel. Der findes ingen anden tekstkolonne i dette layout.

## Control-bytes i familien

Fra sweep af 0x00-0x3F:

| Byte | Virkning **[målt]** |
|---|---|
| `05` `06` `07` | tænder LED 5, 6, 7 |
| `0B` `0D` | skriver tekst midt på skærmen |
| `0C` `0E` `26` `2E` | rydder skærmen |
| `15` | tilføj listepunkt (se ovenfor) |
| `16` | som 0x15, men hvid skrift |
| `17` `18` | lille overskrift plus grå popup |
| `19` | popup med to linjer |
| `1A` | popup, højrestillet tekst |
| `1B` `1E` | popup, flere linjer efter hinanden |
| `1F` | rydder overskrift, skriver med lille skrift |
| `22` | popup med halvfyldt søjle og pile |
| `23` | tom søjle med pile |
| `25` | en slags velkomstskærm |
| `27` | farvepalet øverst til venstre — **ikke udforsket** |
| `28` | sidetitel |
| `29` | rydder siden, efterlader klaver-ikon |
| `2A` | stort klaver-ikon |
| `2F` | keyboard med farvemarkering, ligner keyrange — **ikke udforsket** |
| `31`-`34` | MainStage-liste-widgeten, også her |

- **[målt]** 0x32, 0x33 og 0x34 virker kun hvis 0x31 har kørt først.
- **[målt]** 0x31 trækker MainStage-skillelinjen og footer-båndet med sig,
  også inde i denne familie.
- **[målt]** De to liste-widgets (0x15 og 0x31) udelukker hinanden. Den sidst
  tegnede overtager skærmen. Man kan ikke få AL-punkter til venstre og
  0x32-kolonne til højre samtidig.

## Øvrige control-bytes i DAW-familien

Fra sweep af 0x00-0x3F **[målt]**:

| Byte | Virkning |
|---|---|
| `02` `04` `08` | tænder LED 2, 4, 8 |
| `14` | overskrift |
| `15` | listepunkt, fuld bredde (kræver 0x28 først) |
| `16` | som 0x15, men slukker LED og skubber indholdet op |
| `19` | popup |
| `1C` | encoder-visning |
| `1D` | fader-visning |
| `1E` | pad-/generel popup |
| `25` | velkomstskærm |
| `27` | farvevælger |
| `28` | sidetitel, initialiserer listen |
| `29` | gråt keyboard-ikon øverst med overskrift |
| `2A` | gråt keyboard-ikon på midten |
| `2F` | keyboard-tegning til keyrange |

---

# CC i Arturia-mode

Her gælder en helt anden mapning end i DAW-mode. Blandes de to sammen, ligner
det en firmwarefejl.

## Skærmknapper

**[målt]** Kanal 1, momentan:

| Knap | CC |
|---|---|
| 1 | 60 |
| 2 | 61 |
| 3 | 62 |
| 4 | 63 |
| 5 | 56 |
| 6 | 57 |
| 7 | 58 |
| 8 | 59 |

De to grupper ligger ikke i forlængelse af hinanden: 1-4 er 60-63, 5-8 er 56-59.

## Fadere og encodere

**[målt]** Kanal 1. Rækkefølgen er uregelmæssig — læs tabellen, gæt ikke.
Encoderne sender absolutte værdier 0-127, ikke relative trin.

| Nr. | Fader | Encoder |
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

Faderne bruger 72, 73, 75, 79-83 og 85. Hullerne i rækken — 74, 76, 77, 78 og 84 —
er ikke tilfældige: 74, 76 og 77 er encoder 1, 3 og 4.

---

## Pad-banker og User-mode-konfiguration

De øvrige pad-banker er **lag over de samme 12 fysiske pads**. Der er ingen
pad 13: pad-numrene er 1-12 i alle banker, og LED-id'erne er de samme.

### Bankernes noder

**[målt]** Bank-noderne kommer ud ad den **fysiske MIDI-port** på kanal 10, ikke
DAW-porten. Bank A starter på node 36, B på 48, C på 60, D på 72.
(B og C er udledt af mønsteret, ikke målt.)

MCC's egen nummerering er kromatisk: Pad 1 = 36, Pad 2 = 37, ... Pad 12 = 47 i
bank A. Ableton viser rækkerne omvendt (drum rack-layout, C nederst til
venstre), så det man ser som pad 9 nederst til venstre er MCC's Pad 1.

**[målt]** Bank D's rækkefølge, som pads sidder fysisk:

    pad  9 10 11 12  =  node 72 73 74 75
    pad  5  6  7  8  =  node 76 77 78 79
    pad  1  2  3  4  =  node 80 81 82 83

Altså C4 nederst til venstre op til B4 øverst til højre.

### Konfigurations-familien `21 03 40` — hvor farverne bor

Kortlagt 5/9-2026 ved at diffe to MCC-gemninger der kun adskilte sig ved bank
D pad 1's farve (gul mod hvid). Præcis **én byte** var forskellig.

    F0 00 20 6B 7F 42 21 03 40 <lo> <hi> <værdi> F7    skriv én byte
    F0 00 20 6B 7F 42 20 09 40 <lo> <hi> F7            læs én byte
    F0 00 20 6B 7F 42 21 09 40 <lo> <hi> <værdi> F7    enhedens svar på læsning
    F0 00 20 6B 7F 42 06 03 F7                         commit / gem
    F0 00 20 6B 7F 42 24 03 20 <idx> <side> <tekst> F7 navne ("Pad 4", "Fader 1")

Adressen er `lo + hi * 128`.

**Pad-post, 12 bytes, stride 0x0C:**

    <farve>, 01, <node+8>, <node>, <padnr>, 0A, 00, 00, 7F, 00, 7F, 00

Hver pad har fire poster, én pr. bank. Pad 1 ligger på 0x00D3 (bank A, node 36),
0x00DF (B, 48), 0x00EB (C, 60), 0x00F7 (D, 72).

**[målt]** Farven er **ét palet-indeks**, ikke en RGB-trippel som i
`00 02 03`-familien. Kendte værdier: `0x28` = standard/hvid, `0x05` = gul.
Resten af paletten er ikke kortlagt — den nemmeste vej er at sætte pad 1 til en
farve i MCC, gemme, og læse indekset ud af loggen.

**[målt]** En skrivning slår igennem med det samme, også uden commit, og
overlever et mode-skift. Men den river enheden over i User mode, så den kan
**ikke** bruges til dynamisk farve-feedback under spil.

### Hvad bank D så kan bruges til

Farver sættes én gang i MCC og bliver stående — statisk, ikke feedback.
MIDI'en på kanal 10 kan fanges med et MIDI-spor på den fysiske port, og
skærmen kan give popups med navne. Man får altså feedback, bare på displayet
i stedet for i pad'en.

---

## Afsendelse i praksis

Fem ting der ikke står i nogen protokoltabel, men som afgør om det virker:

1. **Én besked pr. kald.** **[målt] 20/8-2026:** sendes pakken som en række rå
   enkelt-bytes, som modtageren selv skal samle, tabes 2-4% af pakkerne. Sendt
   som én samlet besked: 111 tryk uden ét tab. Det var hele forklaringen på
   "hver 30.-40. gang opdaterer skærmen ikke", og det forklarer også hvorfor
   Arturias eget script aldrig fejlede over samme kabel og port.

2. **Pause mellem skærmpakker.** **[kode]** To pakker i familie `00 02 04` sendt
   umiddelbart efter hinanden kan race på selve enheden, så den ene tabes.
   Riggen kører alle skærmpakker gennem en kø med 60 ms mellem hver. LED-pakker
   (`00 02 03`) sendes uden om køen og har ikke vist samme problem.

3. **Ikke for mange på én gang.** 12 pakker samtidig er i orden, 36 fryser
   enheden. Se LED-afsnittet.

4. **Kun ren ASCII.** **[kode]** Skærmen får kun tegn i intervallet 32-126.
   æ/ø/å skal translittereres (ae/oe/aa), og typografiske anførselstegn og
   tankestreger skal erstattes, ellers ryger tegnet — eller pakken.

5. **Tekstlængder.** Riggen klipper footer-etiketter ved 12 tegn og
   listelinjer ved 18. Det er valgte, afprøvede værdier, ikke målte maksima.

---

## Sammenfatning: valget mellem layouts

| | 0x28 + 0x15 | 0x31-listen | 0x15 i AL-familien |
|---|---|---|---|
| Linjer | 7 | 5 | 7 |
| Kolonner | 1, fuld bredde | 2 | 1, med ikonpanel |
| Skillelinje | nej | ja | nej |
| LED i øverste række | gratis | koster 2 linjer (5 → 3) | gratis |
| Familie | `00 02 04` | begge | `00 01 04` |

**0x28 + 0x15 i DAW-mode er det rigtige valg.** 7 linjer i fuld bredde, lys i
begge knaprækker, og CC-mapningen er den riggen allerede bruger.

## Blindgyder

- Farve uden type-felt i familie 02 — ignoreres.
- `line_number` som middel til lavere footer-række — ingen virkning.
- Kolonne-etiketter fjernet for at få plads — båndet bliver.
- Størrelses- og antalsfelter som middel til flere linjer — ingen virkning.
- Familie 02-appends oven på en AL-side — ignoreres.
- `21 11 40 02 00 <n>` som mode-skift — ingen virkning.
- `0x15` felt 04 som cursor-kommando — det er felt 03.
- `0x15` felt 02 som farvestyring på markøren — sweepet 0-20, ingen effekt.
- Felt 02 i 0x28 sat til listens fulde længde mens kun de synlige punkter
  sendes — så regner firmwaren markøren i en forkert ramme.
- `00 02 03`-LED-pakker som middel til at farve pads i bank D — ignoreres helt;
  banken tegner selv ud fra User-programmet.
- Id 94 og opefter som "pad 13" — der findes ingen pad 13. Bankerne deler de
  samme 12 dioder.
- `00 01 03` som formodet Arturia-pendant til pad-LED-familien — ikke afprøvet
  til bunds, men blev overhalet af fundet af konfigurations-familien.
- Position-i-række som formel for LED-id ("id = position i CC-målerækkefølge")
  — den rigtige formel bruger den FYSISKE knap-rækkefølge.
- 0x1C (encoder-layout) til fader-visning — værre end 0x1D.
- Fejlsøgning af "manglende fader-opdatering på skærmen" — det er kapacitiv
  hudkontakt, ikke protokol.

## Ikke udforsket

- Control-bytes over 0x3F i begge familier.
- 0x27 (farvepalet) og 0x2F (keyboard-visning).
- Lister over 127 punkter: kræver at hi-byten i længde- og indeksfelterne
  tages i brug. Aldrig prøvet.
- Rec/Redo/Undo/Quantize/Saves LED-id — kun formel, ikke målt.
- Pad-LED 82, 90 og 91 — kun formel, ikke målt.
- Palet-indekserne i `21 03 40`-farvebyten ud over 0x28 og 0x05.
- Resten af konfigurations-billedet: hvad `21 09 40`-regionen indeholder, og
  hvad de øvrige bytes i pad-posten (`0A`, de afsluttende `7F`) betyder.
- Bank B og C's noder er udledt af mønsteret (48 og 60), ikke målt.
- Om læsekommandoen `20 09 40` også dækker `03`-regionen, så hele
  konfigurationen kan dumpes uden MCC.
- Om popup-farvefeltet (0x1E felt 04) rent faktisk virker.
