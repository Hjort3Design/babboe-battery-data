# GWA ibo-COP2 EEPROM Register Map

Source of truth for the 256-byte EEPROM (I2C addr `0x50`) fitted to the ibo-COP2
BMS board. This consolidates and reconciles two prior, independently-evolved
decoders in this repo — `cop2_logger.py` and `GWA_Battery_Diag/GWA_Battery_Diag.ino`
— which disagree on several points below. Where they disagree, both readings are
shown and the field is marked `CONFLICT` until resolved by a probe session
(see `PROBE_PROTOCOL.md`).

## Confidence key

| Tag | Meaning |
|---|---|
| `CONFIRMED` | Behavior verified against a known real-world action (a controlled probe), value makes physical sense, and is stable across multiple packs. |
| `HYPOTHESIS` | A plausible interpretation exists (naming, encoding, scale) but has not been confirmed by an isolated probe. |
| `CONFLICT` | The two existing decoders disagree on encoding or meaning for this address. |
| `UNKNOWN` | Byte(s) observed to be non-constant or structured, but no working hypothesis yet. |
| `CONSTANT` | Never observed to change across any captured dump — likely factory-set (serial/date/model/calibration constant), not a live counter. |

## Common encodings seen in this map

- **Raw byte** — single byte, direct value.
- **LE16+XOR** — 4-byte record `[lo, hi, xor, 0xFF]`, value = `(hi<<8)|lo`, `xor` byte should equal `lo^hi` (integrity check, not a real 3rd data byte). Padding byte at `+3` is typically `0xFF`.
- **BE16+XOR** — same record shape but value = `(hi<<8)|lo` with `hi` stored first (`[hi, lo, xor, 0xFF]`). Cannot be told apart from LE16+XOR by inspection alone when `hi==lo^xor` is symmetric — must be resolved by probing a field whose real-world value you know.
- **mV table** — 4-byte groups `[vA, vB, xor, 0xFF]` repeated, each of `vA`/`vB` is `raw + offset` millivolts, `0xFF` in a slot means "tap not populated."

## ibo-COP2 EC error codes (from official manual, `ibo-COP2 Charger.pdf`)

This is the authoritative version - more complete than the abbreviated 8-entry list read
off the physical charger label earlier (that one was missing EC#9 and EC#10, and used
shorter wording). LED columns: POWER, CAP1-4 (battery capacity, green), CAL (white), COP
(yellow), ERR (red).

| EC | Meaning | POWER | CAP1 | CAP2 | CAP3 | CAP4 | CAL | COP | ERR | Impact |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | No charging current | ON | ON | OFF | OFF | OFF | OFF | OFF | ON | Failed to operate |
| 2 | Communication error | ON | OFF | ON | OFF | OFF | OFF | OFF | ON | Failed to operate |
| 3 | Bad cell(s) | ON | ON | ON | OFF | OFF | OFF | OFF | ON | Failed to operate |
| 4 | COP2 over temperature >80°C | ON | OFF | OFF | ON | OFF | OFF | OFF | ON | eBike still working |
| 5 | Connection problem in MX216 connector, or R37 30Ω failed | ON | ON | OFF | ON | OFF | OFF | OFF | ON | Failed to operate |
| 6 | Cell temperature >45°C or <0°C, or R37 NTC failed | ON | OFF | ON | ON | OFF | OFF | OFF | ON | Failed to operate |
| 7 | Cells unbalanced / cell difference >0.4V | ON | ON | ON | ON | OFF | OFF | OFF | ON | eBike still working |
| 8 | Individual cell voltage calibration failed (cell voltage ≤ ±200mV) | ON | OFF | OFF | OFF | ON | OFF | OFF | ON | Failed to operate |
| 9 | Current sensor failed | ON | ON | OFF | OFF | ON | OFF | OFF | ON | eBike still working |
| 10 | COP2 self-check on individual circuit loop (on connecting COP2 & charger) | ON | OFF | ON | OFF | ON | OFF | OFF | ON | eBike still working |

Note EC#10 is not really a fault - it's the normal self-check state on power-up (matches
the charger label's "5 second self-diagnostic, all 8 LEDs flash left to right" description
already seen). EC#3/#5/#6/#8 relate directly to fields already in this register map -
`0xE0`-`0xFF` cell voltages (EC#3, #6, #8) and the capacity/threshold fields (EC#1). Worth
testing whether `0x00`'s value changes to something EC-code-shaped when the physical LEDs
show one of these states, which would finally pin down `status_byte`'s bit meanings.

## Battery Health Index (BHI) grade table (from manual, for reference)

Relevant context for `0x60` (capacity threshold) and the `gradeFromPct()` logic already in
`GWA_Battery_Diag.ino`:

| Grade | Capacity after aging | POWER | CAP1-4 | CAL | COP | ERR |
|---|---|---|---|---|---|---|
| A | ≥80% | ON | all ON | ON | OFF | OFF |
| B | ≥70% | ON | 1-3 ON, 4 OFF | ON | OFF | OFF |
| C | ≥60% | ON | 1-2 ON, 3-4 OFF | ON | OFF | OFF |
| D | ≥50% | ON | 1 ON, 2-4 OFF | ON | OFF | OFF |
| E | <50% | ON | all OFF | ON | OFF | ON |

This matches `GWA_Battery_Diag.ino`'s existing `gradeFromPct()` thresholds (80/70/60/50)
exactly - good independent confirmation that the firmware's grading logic is correct and
sourced from the same spec.

## Physical access points on the charger board (GWA-COP2-2018-1109)

Confirmed by direct inspection/continuity check, not inference - useful for anyone wiring
up a probe on this board without re-deriving it:

- **`TX1`/`RX1`/`G` test pads** (next to the ICSP header) and the **`G RX TX SW+ CH_IN`
  keyed 5-pin connector** (near date code `1902`) **are the same net** - confirmed by
  continuity check 2026-08-11. The keyed connector is the far easier attachment point
  (proper mating connector possible, no soldering to tiny pads needed). `SW+` likely ties
  to the physical self-diagnostic button sequence described on the charger's printed label
  ("1st Step: Initiate COP2 / 2nd: Charger / 3rd: Battery"); `CH_IN` is unconfirmed but
  plausibly "charge input."
- **ICSP header** (`ICSPCLK`, `ICSPDATA`, `G`, `+5V`, `-MCLR`) - PIC16F1947 programming/debug
  interface. Only useful with a PIC programmer (not currently available); would allow a
  full firmware dump if the chip isn't code-protected.
- **`J7` header** (`CS`, `SO`, `G`, `SI`, `SCL`) - SPI test points near a small SOIC-8
  chip marked `T4534` / `CS7208P` (unidentified part, no datasheet found; board position
  next to switching MOSFETs `Q10/Q11/Q14/Q16` and diode array `D29-D44` suggests it may be
  a gate-driver/PWM controller for the charge power stage rather than a memory chip -
  unconfirmed, continuity check between `J7` and this chip's pins would settle it).
- **Two `CD4052BM` dual 4-channel analog multiplexers** on the board (16 channels total) -
  originally read as a clean physical-layer match for 16 independent cell-voltage taps at
  `0xE0`-`0xFF`. **Revised 2026-09-06:** all packs are confirmed **32.85V nominal, 9S**
  (9 series cells - `32.85 / 9 = 3.65V/cell`, a clean standard Li-ion nominal, not the 10S
  this project had been assuming). **Officially confirmed** via Babboe's own `ibo-COP2`
  charger datasheet (ManualsLib "Babboe Ibo-R37 User Manual"): charger output rated
  **37.8VDC max** = exactly `9 x 4.2V` full-charge - and that same manual explicitly covers
  the ibo-R45 too, so the whole R37/R45/(R50) lineup shares this 9S topology, with capacity
  differences coming from parallel cell count, not series count. A 9-cell series stack only
  needs 9-10 real tap points (matching `cop2_logger.py`'s `V0`-`V9` naming), not 16 - so the
  clean "16 mux channels = 16 taps" story no longer holds. The 16 raw values at `0xE0`-`0xFF`
  are more likely some mix of redundant/dual-bank measurements over the same ~9 cells, or
  partly-unused channels, rather than 16 genuinely distinct series cells. See revised notes
  on that row below - `multimeter_taps` is now more important than ever for sorting this out.
- **EEPROM itself is inside the battery pack**, not on this charger board - a single
  8-pin chip, I2C address `0x50`. This is the one that matters most for the register map;
  everything else on this list is charger-side and secondary.

## Map

| Addr | Size | Field (cop2_logger.py) | Field (GWA_Battery_Diag.ino) | Encoding | Status | Notes |
|---|---|---|---|---|---|---|
| `0x00` | 1 | `status_byte` | `status_byte` | raw | `HYPOTHESIS` | Seen `0xC0`, `0x01`, `0x7A` (all reads), and now a live-captured **write** of `0x10` (2026-08-13), landing right at the start of the self-test sequence, before the `0x80`-`0xDF` log-block writes begin. Plausible candidate: a "self-check in progress" status code (matches EC#10 - "self-check on individual circuit loop, on connecting COP2 & charger" - which the manual explicitly says isn't a real fault). The other values seen so far were all reads, likely post-self-test steady-state. GWA treats `>=0xE0` as high EC#2 risk (fault/error state), otherwise assumed "normal". No confirmed bit-level mapping yet. **Full EC fault taxonomy now known from the official ibo-COP2 manual** (`ibo-COP2 Charger.pdf`, section 3.5/"Error codes on the COP2") - see the dedicated table below. **Action:** force each EC condition where practical (e.g. disconnect a cell tap for EC#3, block airflow for EC#4) and dump `0x00` after each to build the bit/value mapping against this table. **Update 2026-10-04:** the two newer packs store this as a full `[status, FF, XOR]` record (`02 FF FD`, `01 FF FE`, both XOR-valid); Bat:001's `C0 FF FF` is not XOR-valid, so older firmware may not checksum it. |
| `0x01`-`0x0F` | 15 | - | - | raw | `UNKNOWN` (reopened) | **Reopened 2026-08-13:** previously always `0xFF` in every static dump, but a live capture caught the charger *writing* `0x01`=`0x00` and `0x02`=`0x01` during a self-test sequence (same event as the `0x00`=`0x10` write noted above, same timestamp). Small sequential values written alongside a status byte suggests `0x00`-`0x02` may be a short status/stage record (e.g. a self-test step counter) rather than `0x00` being a lone byte with 15 bytes of padding after it. Static dumps only ever caught it at rest (`0xFF`) because the EEPROM's steady-state value outside an active self-test may genuinely be `0xFF` - worth watching `0x01`/`0x02` specifically during the next few live captures to see if they settle back to `0xFF` after self-test completes, or hold a value. **New sub-finding 2026-09-06:** `0x0C`-`0x0E` (part of this same 15-byte span) is a valid LE16+XOR record (`46 32 74` = 12870) on the R45 pack (identity `06 06 00`) but factory-`FF` on both R37 packs - the first field seen so far that looks genuinely model-specific rather than per-pack or per-firmware-version. Meaning open. |
| `0x10` | 4 | ~~`plug_ins`~~ | ~~`charge_events`~~ | LE16+XOR | `UNKNOWN` (meaning reopened) | Encoding agreed (LE16+XOR). Previously observed decreasing (325->298), ruling out a simple monotonic counter. **New 2026-08-14:** live-captured a clean `+1` (`364`->`365`, both XOR-valid) landing in the exact same atomic write group as the full calibration event (`0x30`/`0x34`/`cal_stat1-3`/cell tap). This is the first time this field's movement has been pinned to a specific, known event - worth watching whether it *only* moves at calibration completion (which would make "calibration event counter" the best current label) or also moves at other times (ruling that back out). **Update 2026-10-04:** the "only moves at calibration" idea is refuted. Bat:001 went 299 -> 373 (+74) between 2026-08-13 and 2026-10-04 with only one calibration (`0x2C` 1 -> 2) in that time, while `0x50` rose +97. Best current label: a charge-event counter that moves on most, but not all, connects. Bat:003: Recode set it to 0 and one short charge left it at 0. |
| `0x14` | 4 | - | - | LE16+XOR | `UNKNOWN` (populated) | **Update 2026-09-06:** Bat:003 shows this as a valid XOR-passing record (`08 02 0A FF` = 520) - not blank as assumed from Bat:001's dumps, just previously never seen populated. **Same day:** the R45 pack (identity `06 06 00`) reads the *exact same* 520 - shared between an R37 and an R45 pack, so not a model-tier code like `0x24`/`0x28`. Bat:001 (R37) still reads blank here despite being the same model as Bat:003 - doesn't split cleanly by model OR by pack, so likely tracks something else entirely (firmware/calibration-history version? see the same pattern on `0x3C` below). Meaning still open. |
| `0x18` | 3-4 | `date_field`/manufacture guess | `identity_field` (`ADDR_IDENTITY`) | raw 3 bytes | `HYPOTHESIS` (cross-pack data in) | **Update 2026-09-06:** a second physical pack (Bat:003) reads `07 03 04`, vs pack #1's `07 1A 1D` - different trailing 2 bytes, but the **same leading byte `0x07`** on both. Best current read: leading byte is a shared model/family constant (both packs rate 500 units), trailing 2 bytes are a per-unit serial-like value. Not yet `CONFIRMED` - still want a 3rd pack to be sure the leading byte isn't coincidence. |
| `0x1C` | 4 | ~~`full_cycles` (LE16+XOR)~~ | `stored_capacity` - **BE16+XOR** (`ADDR_STORED_CAP`, `readXOR_BE`) | **BE16+XOR** | `CONFIRMED` | **Resolved 2026-08-11** from a fresh BAT:001 dump (bytes `01 B1 B0`): LE reading gives 45313 — over 9000% of the 500-unit rated capacity (`0x24`), physically nonsensical. BE reading gives 433, i.e. 433/500 = 86.6% of rated — a sane stored-capacity percentage for a pack with only moderate cycle history. GWA's `stored_capacity`/BE16+XOR interpretation is correct; `cop2_logger.py`'s `full_cycles`/LE decode for this address should be fixed. |
| `0x20` | 4 | - | - | LE16+XOR? | `UNKNOWN` (new) | **New 2026-08-14:** first-ever observed write to this address, live-captured in the same atomic group as a full calibration event - value `00 00 00` (all zero). Never previously listed in either decoder; sits in the gap right after `0x1C`'s 4-byte group. Only one data point so far (a zero, right after calibration) - not enough to say what it tracks yet. |
| `0x24` | 4 | `rated_capacity` | `rated_capacity` (`ADDR_RATED_CAP`) | LE16+XOR | `CONFIRMED` (model tier code, not literal Wh) | Both tools agree: value scales with pack size (500 units seen on smaller pack, 600 on larger). **Settled 2026-09-06:** Bat:003's physical label reads **`R37 / 11.4Ah`** - directly confirmed R37, ruling out the earlier R50 theory. But 11.4Ah x 36V = 410.4Wh, which doesn't cleanly convert from raw `500` under *any* tidy factor - so raw `500` was never a literal Wh or Ah encoding. Best model: **`0x24` is a fixed capacity-class/model-tier code** (`500`=R37-family, `600`=R45-family), shared by every pack of that tier regardless of the actual printed Ah/Wh, which can vary a bit unit-to-unit. **Confirmed same day** from a scan log CSV: a third pack (identity `06 06 00`) reads raw `600` - matching the older `cop2_logger.py` note ("600 on larger pack") rather than `GWA_Battery_Diag.ino`'s own hardcoded `RATED_R45 = 596` constant, which now looks like an approximation, similar to the `374/500` Wh-display formula also being a rough approximation rather than an exact per-unit conversion. **Update 2026-09-11:** `GWA_Battery_Diag.ino`'s "Recode after cell replacement" feature now has a live-write path for this field (`doRecode()`'s optional `retierCode` param, server-validated), restricted to only the two confirmed codes (`500`/`600`); R50's raw code remains unconfirmed and is not writable through this path. |
| `0x28` | 3-4 | `manuf_info` - raw, unknown | `cal_cycles` - **BE16+XOR** (`ADDR_CAL_COUNT`) | BE16+XOR | `CONFIRMED` (constant per model tier, not universal) | **Update 2026-08-11:** encoding confirmed BE16+XOR (bytes `34 21 15`, decodes to 13345), never changed across many sessions or even a real live calibration on pack #1 - ruling out GWA's "cal_cycles" (live counter) label. **Confirmed 2026-09-06:** Bat:003 (a second, independent R37 pack) reads the exact same `34 21 15` / 13345. **Revised same day:** a third pack (R45, identity `06 06 00`) reads `A0 28 88`, which decodes BE-consistently to **41000** - genuinely different from the R37 packs' 13345, so this is a **per-model-tier constant, not one universal value across the whole product line**. (A scan-log CSV briefly showed this same R45 pack as `10400` - that's a stale artifact from an older firmware version that decoded this field with the wrong byte order before a mid-project fix to BE; recomputing directly from the raw hex with the current, consistent BE convention gives 41000. Lesson: always re-derive from `raw_hex` rather than trusting an old decoded column, since decode logic has changed over the project's history.) |
| `0x2C` | 4 | - | - | LE16+XOR | `HYPOTHESIS` (strong: calibration count) | **New 2026-08-14:** first-ever observed write to this address, live-captured in the same atomic calibration group as `0x20` above - value `02 00 02` (XOR-valid, decodes to 2). Never previously listed in either decoder; sits in the gap right after `0x28`'s group. A small value (2) written alongside `0x20`'s zero, right at calibration completion - possibly a small counter or flag pair with `0x20`, but only one data point so far. **Update 2026-10-04 - probably the calibration count (upgraded to `HYPOTHESIS`, strong):** 0 on Bat:003 (user confirms it has never done a full CAL cycle), 0 on Bat:001 in June before its first calibration, 1 after it, 2 after the 2026-08-14 calibration, and 1 on the R45 pack (which has calibration data). Consistent on all three packs. Next check: Bat:003 should read 1 after its first CAL cycle. |
| `0x30` | 4 | `addr30_value` - raw, unknown, logged raw in CSV | `cal_measured` - **LE16+XOR** (`ADDR_CAL_MEASURED`) | LE16+XOR | `CONFIRMED` (model revised) | **Confirmed 2026-08-14** via a live-captured full calibration cycle: value changed for the first time ever observed, `7456` -> `6666`, in the same atomic write group as `cal_stat1-3` and a cell-tap voltage populating. **Revised 2026-09-06:** a second pack (Bat:003) that has *never* had a live calibration (cal_stat1-3 and all cell-voltage tables still factory-`FF`) nonetheless has a real, XOR-valid `0x30` value (3936, mirrored into `0x40`/`0x44`). So `0x30` isn't exclusively written by the live "field calibration" event - it can also carry a **factory-set initial measurement** from end-of-line testing. The "5 fields written atomically" model still holds for a live *field* recalibration; it just doesn't mean `0x30` is only ever touched by that path. |
| `0x34` | 4 | not tracked | not decoded | LE16+XOR | `HYPOTHESIS` (mirror theory incomplete) | Byte-for-byte mirror of `0x30` in pack #1's 2026-08-14 live calibration (both changed to `6666` together). **Revised 2026-09-06:** pack #2 (Bat:003) breaks the simple "always mirrors" rule - `0x30`=3936 (factory-set, see above) but `0x34` is still factory-`FF`, completely unpopulated. Better model: `0x34` gets written only by an actual **live field calibration** event (same one that populates `cal_stat1-3` and the cell tables), not by whatever process pre-loads `0x30` at the factory. Downgraded from `CONFIRMED` pending a pack that's had at least one real field calibration to re-verify the mirror behavior in isolation. |
| `0x38` | 3-4 | `field38` - raw, small value (`01 00 01`) | not decoded | raw | `UNKNOWN` | Only cop2_logger notes this; looks like it could itself be a 2-byte+XOR record (`01`,`00`,`01`=`01^00`) with value=1 - consistent with a boolean/flag field. Worth adding LE16+XOR decode and watching for transitions 0->1. |
| `0x3C` | 4 | - | - | LE16+XOR | `UNKNOWN` (populated) | **Update 2026-09-06:** Bat:003 shows this as a valid XOR-passing record (`3C 28 14 FF` = 10300) - not blank as assumed from Bat:001's dumps. **Same day:** the R45 pack reads a close-but-different `12300`. Bat:001 (R37, same model as Bat:003) is still blank here - same "populated on Bat:003 + R45, blank on Bat:001" split as `0x14` above, reinforcing that whatever gates these two fields isn't simply model tier. Meaning still open. |
| `0x40` | 4 | ~~`addr40_flags`~~ | ~~`session_mah`~~ | LE16+XOR | `HYPOTHESIS` (revised, see finding below) | **Update 2026-08-11:** confirmed to move in both directions - jumps up to exactly match `0x30` when a calibration event completes, then fluctuates *down and back up* with normal use (7456->7018->7240 across 3 dumps, no calibration in between). Best current read: a **live gauge tracking charge/discharge relative to the `0x30` calibration ceiling**, not a countdown timer or lifetime totalizer. See "Major structural finding" / "Chronological reconstruction" sections below for the full evidence chain. |
| `0x44` | 4 | `addr44_mirror` - mirror of `0x40` | not separately decoded | LE16+XOR | `HYPOTHESIS` | cop2_logger notes this mirrors `0x40` in observed captures - if that holds, it may be a redundant/backup copy (common in EEPROM designs to survive a torn write) rather than an independent field. |
| `0x48` | 4 | - | - | LE16+XOR | `UNKNOWN` (added 2026-10-04) | Read by the charger at every self-test (see 2026-08-13 read sweep). 12000 on Bat:003, 12300 on R45, blank on Bat:001. |
| `0x4C` | 4 | - | - | LE16+XOR | `UNKNOWN` (added 2026-10-04) | Read by the charger at self-test. 1000 on both Bat:003 and R45, blank on Bat:001. |
| `0x54` | 4 | - | - | - | `UNKNOWN` (added 2026-10-04) | Blank on every pack so far and not in the charger's read sweep. Possibly unused. |
| `0x58` / `0x5C` | 8 | - | - | LE16+XOR | `UNKNOWN` (added 2026-10-04) | Both read by the charger at self-test. Only the R45 pack has values (4100 / 3800). |
| `0x50` | 4 | ~~`charge_minutes`~~ | `session_counter` (`ADDR_SESSION_COUNT`, LE16+XOR) | LE16+XOR | `CONFIRMED` | **Confirmed 2026-08-13** via live logic-analyzer capture: the charger read `0x50` as `1511` early in a self-test sequence, then ~1.2s later *wrote* `1512` back to the same address - a live, caught-in-the-act `+1` increment within a single connect/self-test event. `cop2_logger.py`'s `charge_minutes` label is confirmed wrong; `session_counter` is correct. See dated section below for the full transaction capture. |
| `0x60` | 4 | `field60` (LE16+XOR) | `cap_threshold` (`ADDR_CAP_THRESHOLD`, LE16+XOR) | LE16+XOR | `HYPOTHESIS` (percentage theory weakened) | Both tools interpret this as a minimum-capacity threshold - likely the EC#2 low-capacity fault threshold. **Update 2026-09-06:** pack #1 read 358 (~71.6% of the 500-unit rated capacity); pack #2 (Bat:003, never calibrated) reads only 155 (~31%). That's too wide a spread for a fixed "~71% of rated" percentage rule - either this value is itself set at calibration (and pack #2's is a factory placeholder, not a real threshold yet), or the threshold isn't rated-capacity-relative at all. Needs a calibrated pack + an actual EC#2 trigger to pin down. **Update 2026-10-04:** on Bat:001 this climbs slowly and never falls: 356 (June) -> 358 (Aug) -> 365 (Oct). That looks more like a lifetime counter (maybe full-cycle equivalents) than a threshold. Recode does not reset it (Bat:003 still 155 after recode). |
| `0x64`-`0x73` | 16 | - | - | LE16+XOR (4 records) | `UNKNOWN` (structure found, cross-pack data in) | **Update 2026-09-06:** long-standing fully-unknown block turns out to be fully populated, XOR-valid data on Bat:003: `0x64`=6000, `0x68`=7000, `0x6C`=8000, `0x70`=9000 - a perfectly clean 1000-unit step sequence (corrected from an earlier arithmetic slip that mis-read this as 6000/6968/7936/8996 - re-verified directly against the raw hex). Bat:001 (R37) has this whole block still factory-`FF` (blank). A third pack, an R45 (identity `06 06 00`), shows `7400, 8400, 9600, 10700` - close in magnitude but not the same clean step size. Best current read: Bat:003's suspiciously round, exact-1000-step numbers look like an unwritten **factory placeholder/template** (consistent with Bat:003 never having had a real field calibration), while the R45 pack's slightly-uneven numbers may be genuine post-calibration measurements (that pack's `cal_stat1-3` are populated, unlike Bat:003's). Still open - needs a freshly-calibrated R37 to see if its 0x64-0x73 changes from the round placeholder pattern once a real calibration happens. |
| `0x74` | 4 | not decoded | `cal_stat1` (LE16+XOR) | LE16+XOR | `HYPOTHESIS` (encoding confirmed, live-captured) | Populated `1B 19 02`, XOR-valid, decodes to 6427 - now seen twice (a static dump on 2026-08-11, and live-captured as part of the full atomic calibration write on 2026-08-14, same value both times). Structure solid, semantic meaning still open. |
| `0x78` | 4 | not decoded | `cal_stat2` (LE16+XOR) | LE16+XOR | `HYPOTHESIS` (encoding confirmed) | Populated in the same 2026-08-14 live calibration capture, decodes to 16448 - note this differs from the `35 35 00` (13621) seen in the 2026-08-11 static dump, meaning this field **does change between calibration events**, unlike `cal_stat1`/`cal_stat3` which repeated identical values both times. Worth flagging as the most likely of the three to encode something calibration-specific rather than a fixed status code. **Update 2026-10-04 - possible cell 9 top-of-charge voltage:** both bytes of this record are always equal (`35 35`, `40 40`, R45 `36 36`), i.e. one byte stored twice. Read with the cell table's `+4000` offset it gives 4053 / 4064 / 4054 mV - a normal top-of-charge cell voltage. The `0xE0` table only holds 8 cells and the pack is 9S, so this may be the 9th cell. Unconfirmed; check with a multimeter. |
| `0x7C` | 4 | not decoded | `cal_stat3` (LE16+XOR) | LE16+XOR | `HYPOTHESIS` (encoding confirmed, live-captured) | Populated `98 98 00`, XOR-valid, decodes to 39064 - close to but not identical to the `9B 9B 00` (39835) seen 2026-08-11; same `hi==lo` pattern both times (possibly two independent single bytes rather than one 16-bit value, as noted before). **Update 2026-10-04 - possible cell 9 bottom-of-charge voltage:** same one-byte-twice pattern (`9B 9B`, `98 98`, R45 `A3 A3`); with the `+3000` offset that's 3155 / 3152 / 3163 mV, right inside the `0xF0` bottom table's range. Pairs with `0x78`. |
| `0x80`-`0xDF` | 96 (24x4) | "log block" - 3 parallel interpretations printed (mV+3000, raw16, ASCII) | `ADDR_HISTORY_LOG` - referenced but not decoded field-by-field in JSON output | 3 groups of 8x 4-byte ASCII+XOR records | `CONFIRMED` (group 1 = printed serial number) | **Update 2026-08-11:** structure resolved to 3 groups of 8 entries (`0x80-0x9F`, `0xA0-0xBF`, `0xC0-0xDF`), each entry `[charA, charB, XOR, 0xFF]`, ASCII-decodable, all sharing a `50 50 00` (`PP`) terminator entry. **Confirmed 2026-08-13:** live-captured the charger writing 8 back-to-back records into this block during self-test - actively written, not a static factory record. **CONFIRMED 2026-09-06 - group 1 meaning solved:** on pack #2 (Bat:003), group 1 decodes to `BB AP TN 52 03 03 9P` -> stripping the terminator/pad gives `BBAPTN5203039`, and the user independently read the exact same string, `BBAPTN5203039`, printed on the pack's physical label. **Group 1 (`0x80`-`0x9F`) is the pack's printed serial number**, 2 ASCII chars per record, XOR-checked, padded with `P` (`0x50`) when the string length is odd, ending in the shared `50 50 00` terminator record. **Cross-checked same day against pack #1** (Bat:001): its group 1 (`BA PD MP 24 R1 26 4P`) decodes to `BAPDMP24R1264`, and the user confirmed this is also its actual printed serial - two independent packs, two exact matches. Groups 2 (`0xA0`-`0xBF`) and 3 (`0xC0`-`0xDF`) follow the identical encoding scheme; on pack #1, groups 1 and 3 happen to share their first 3 entries (`BA PD MP`) while group 2 doesn't, but on pack #2 all three groups instead share just entry 2 (`AP`) - not a consistent cross-pack rule, so groups 2/3's actual meaning (batch/cell-supplier/secondary ID?) is still open. **From a scan-log CSV (2026-09-06):** the R45 pack's group 1 decodes to `BBAPTO4604320` - structurally identical (ASCII+XOR+`P`-pad+`PP`-terminator), predicted to be its printed serial by the same rule confirmed on the other two packs, but not independently checked against this specific unit's physical label yet. `fault_trigger` probe still open for whether a real fault event changes any of this differently than a routine self-test write does. **Update 2026-10-04 - group 2 is probably the ID of the last charger:** the 2026-08-13 captures already showed the charger rewriting exactly `0xA0`-`0xBC` on every AC power-on self-test. Bat:003's group 2 was `PBAPDO0901991` until its first short charge on the user's charger, after which it read `C2N12MP25B140` - identical to Bat:001's group 2, which is always charged on that same charger. Two packs carrying the same value rules out a battery ID; "C2" fits COP2. The user's charger label is unreadable, so to confirm, charge a pack on a different charger and check that group 2 changes. **Group 3 (`0xC0`-`0xDF`)** is never read or written by the charger, so it's a static factory record (BMS board / cell module / paired device?). It can't be the bike unless the bike's battery dock has data contacts - check that first. |
| `0xE0`-`0xEF` | 16 (4x4) | not decoded | `cell_top` - 8 voltages, mV table, offset `+4000` | mV table | `HYPOTHESIS` (structure strongly confirmed, cell-count mismatch open) | **Update 2026-08-11:** a fresh dump has this block fully populated (previously seen mostly/partly `0xFF`) with **all 4 entries passing the `vA^vB=chk` integrity check** - strong structural confirmation of the mV-table encoding. Decoded values cluster tightly at 4076-4091 (i.e. ~4.08-4.09V with the `+4000` offset), consistent with a well-charged pack. **Revised 2026-09-06:** all packs are confirmed 9S (32.85V nominal / 9 = 3.65V/cell), not 10S as previously assumed - see the mismatch note on the `0xF0`-`0xFF` row below, since this changes how these 8 values should be interpreted. Still needs a multimeter cross-check. |
| `0xF0`-`0xFF` | 16 (4x4) | not decoded | `cell_bottom` - 8 voltages, mV table, offset `+3000` | mV table | `HYPOTHESIS` (structure strongly confirmed, cell-count mismatch open) | Same dump: all 4 entries here also pass the XOR check cleanly, decoding to 3156-3164 (~3.16V with `+3000` offset) - a notably different range than `cell_top`'s ~4.08V cluster. **Live-captured 2026-08-14:** tap index 3 (`0xFC`-`0xFE`) written live during a real calibration event - `98 A3 3B`, XOR-valid, decodes to 3152mV/3163mV, right in the same range as the earlier static dump. **Hardware note (2026-08-11):** the charger board has two `CD4052BM` ICs - dual 4-channel analog multiplexers, 8 switched channels each, 16 total - which originally looked like a clean 1:1 match for 16 independent cell taps (8 top + 8 bottom). **Revised 2026-09-06:** all packs are confirmed **9S, 32.85V nominal** (`32.85/9 = 3.65V/cell`, a clean standard Li-ion nominal) - not the 10S this project had been assuming. A 9-cell series stack only needs 9-10 real tap points (matching `cop2_logger.py`'s `V0`-`V9` naming), so 16 raw values can't all be independent series-cell voltages on this pack. Both value ranges (~3.15V and ~4.08V) are individually plausible single-Li-ion-cell voltages (not cumulative stack voltages, which would be far higher than these offsets could encode) - so the more likely explanation is **two redundant/dual-bank measurements over the same ~9 cells** (e.g. two separate mux banks each wired to overlapping or all 9 cells for cross-checking/balance detection), or some channels simply unused/padding on a 9-cell pack, rather than 16 genuinely distinct cells. **Action (still open, now higher priority):** measure real tap voltages with a multimeter (`--voltages` flag) against all 9 physical cells to see which of the 16 raw values actually correspond to real cells vs. redundant/unused channels. |

## Resolved 2026-08-11 (fresh BAT:001 dump)

1. **`0x1C` endianness** - **RESOLVED to BE16+XOR.** LE gave a physically impossible
   value (>9000% of rated capacity); BE gave a sane 86.6%-of-rated reading.
   `cop2_logger.py` needs its `full_cycles`/LE decode at this address fixed to match
   GWA's BE approach and probably renamed to `stored_capacity`.
2. **`0x28`** - encoding confirmed (BE16+XOR, structurally valid), but the value has
   never changed across any dump of this pack, ruling out GWA's "cal_cycles" (live
   counter) label. Likely a fixed factory constant instead - needs cross-battery
   comparison to determine if it's per-pack (serial-like) or model-wide (spec constant).
3. **`0xE0`-`0xFF` cell voltage tables** - structure strongly confirmed (all 8 entries
   now XOR-valid in a fresh dump); still needs a multimeter cross-check to pin down the
   exact tap mapping and confirm the `+4000`/`+3000` offsets.
4. **New finding:** `0x34` mirrors `0x30`, the same pattern already known at `0x44`/`0x40`
   - not previously tracked by either tool.
5. **`0x40`/`0x44`** - re-read as a likely **cumulative lifetime totalizer**, not a
   per-session value (climbed from 0 to 1120 to 7240 across sessions with no resets).
6. **`0x50`** - further evidence favors `session_counter` over `charge_minutes` (a
   +24 delta over what was clearly weeks/months of use is far too small for "minutes").
7. **`cal_stat1`/`2`/`3` (`0x74`/`0x78`/`0x7C`)** - populated with valid data for the
   first time; structure confirmed, meaning still open.

8. **`0x80`-`0xDF` log block** - restructured from "24 unknown slots" to "3 probable
   fixed-format 8-entry records" (`0x80`, `0xA0`, `0xC0`), with records 1 and 3 nearly
   identical and all 3 sharing a `50 50 00` terminator entry. Semantic content still
   `HYPOTHESIS`.

## Major structural finding 2026-08-11: calibration event writes 5 fields atomically

A pair of historical BAT:001 dumps (logged as rows 6-7 in `cop2_log.csv`, "early" and
"the one after") captured the exact moment a calibration cycle appears to have completed:

- `0x30`/`0x34` changed value (11227 -> 7456) - a new measured-capacity reading was written.
- `0x40`/`0x44` jumped from 0 to **exactly 7456** - i.e. synced to match the *new* `0x30`
  value, not incremented independently.
- `cal_stat1`/`cal_stat2`/`cal_stat3` (`0x74`/`0x78`/`0x7C`) went from all-`0xFF` (never
  written) to fully populated, XOR-valid data.
- `0xE0`-`0xFF` cell voltage tables went from all-`0xFF` (never written) to fully
  populated, XOR-valid data.
- Meanwhile `0x1C` stored_capacity, `0x24` rated_capacity, `0x28`, `0x50`, and `0x60` were
  all unchanged across the same pair.

Five fields changing in exact lockstep, with `0x40` landing on precisely `0x30`'s new
value rather than an independently-derived number, is strong evidence these are written
as **one atomic group by a single firmware event** - most plausibly "calibration cycle
completed": it measures fresh capacity (`0x30`), records the cell voltages and balance
status at that moment (`0xE0`-`0xFF`, `cal_stat1-3`), and resets the `0x40`/`0x44` tracker
to the new calibration baseline. This revises the earlier "cumulative lifetime totalizer"
read on `0x40` - it's better described as "capacity accumulated since last calibration,
reset to match `0x30` when a new calibration completes" pending confirmation.

This significantly narrows what the `full_cal_cycle` probe in `PROBE_PROTOCOL.md` needs to
confirm: trigger one calibration cycle, dump immediately before and after, and check that
all 5 fields move together again exactly this way.

**Newest dump (2026-08-11) simplifies the `0x40` picture again.** `0x40`/`0x44` rose from
7240 back up to **exactly 7456** - but this time `0x30` itself was already 7456 and did
*not* change, so this wasn't a fresh calibration event. `0x40` simply climbed back up to
meet `0x30`'s existing value through normal charging. Combined with the earlier
up/down/up sequence, the simplest model that fits all observations so far: **`0x40` is a
live charge-level value in the same units as `0x30`/`0x1C`, capped at the `0x30`
calibration ceiling** - rising while charging, falling while discharging, and only
`0x30` itself changes at an actual calibration event (not `0x40`). This pack was
essentially fully charged relative to its last calibration at the time of this dump.
`0x10` (plug-ins) was unchanged in this same dump (299 -> 299, no drift with no action),
consistent with it being action-driven rather than time-driven, whatever it actually
tracks. `0x50` ticked up by +3, continuing its unbroken non-decreasing streak.

**Follow-up dump (same day) revises this further.** A third BAT:001 dump, with `0x30`/`0x34`
unchanged (still 7456, no new calibration), shows `0x40`/`0x44` **decreasing** from 7456 to
7018 - moving *away* from `0x30` this time, not toward it. Also notable: **`0x10` (plug-ins)
decreased** in this same dump (325 -> 298) - a real drop, not noise (14 bytes changed total,
all XOR-valid). A field that can go down rules out "plug-in count" or any simple monotonic
counter reading entirely.

**Chronological reconstruction (2026-08-11) refines this again.** `cop2_log.csv` timestamps
only reflect when each dump was *logged*, not when it was *captured* on the device -
several historical dumps were pasted in well after the fact. To recover real order, `0x50`
was used as the ordering key (it's the one field never observed decreasing across any pair
of dumps so far, making it the best available proxy for "time"). Sorted by `0x50`, the six
BAT:001 dumps show `0x40` moving **7456 -> 7018 -> 7240** and `0x10` moving
**198 -> 200 -> 325 -> 298 -> 299** - both going down *and back up* across the sequence,
with no calibration event (`0x30` unchanged) between the last two steps. A field that rises
again without a calibration reset isn't a countdown timer counting toward zero; it behaves
like a **live gauge that tracks actual charge/discharge, capped at the `0x30` calibration
ceiling** - rising when the pack is charged, falling when discharged. `0x10` shows the same
up/down/up shape across the same sequence, suggesting it may be a similarly fluctuating
gauge rather than any kind of event counter (contradicting both "plug_ins" and
"charge_events" as candidate names). Meanwhile `0x50` was confirmed strictly non-decreasing
across all 6 dumps in true chronological order - the strongest evidence yet that `0x50` is
the real monotonic session/event counter, and `cop2_logger.py`'s `--compare` and delta
logic should treat `0x50`, not `0x10`, as the reliable "time has passed" indicator.
`cop2_log.csv` has been physically reordered to match this reconstructed chronology.

## Confirmed 2026-08-13: genuine charger self-initiated I2C traffic captured live (logic analyzer)

First real, protocol-level capture of the charger talking to the EEPROM on its own -
previous attempts (passive ESP32 sniffer, first logic-analyzer pass) either caught nothing
or were undersampled (stuck at 20kHz, aliasing any real signal - see that mistake repeated
and fixed below). This one was taken correctly: analyzer set to 2MHz sample rate, tapped
directly on the MX216 connector's SDA/SCL, decoded offline (start/stop/ACK reconstruction
in Python, cross-checked against every byte's XOR integrity check). **Confirmed with the
user that no other master (the old `GWA_Battery_Diag` ESP32) was connected or active during
this capture** - every transaction below is the charger's own PIC, not an external tool.

Two things stood out immediately:

- **The bus is bit-banged, slow (~6kHz effective clock)** - not a fast hardware I2C
  peripheral. This rules out interrupt-timing/CPU-contention as the reason the ESP32
  passive sniffer (`Charger_LiveSniff.ino`) caught nothing in earlier tests; the real
  explanation is simpler: **the traffic is a short, one-shot burst** (~1.3 seconds total,
  once, in a 26-second capture), not continuous polling. Every prior passive-sniff test
  was almost certainly just watching during a window when this event wasn't happening -
  not a wiring or decode-logic failure. **Trigger confirmed: an AC power-cycle of the
  charger, with a battery already connected** (matches the charger label's "5 second
  self-diagnostic on power-up" description, and EC#10's "self-check on connecting COP2 &
  charger" in the manual's error table). Battery plug/unplug alone, with the charger
  already powered, does not appear to trigger it - every earlier passive-sniff test cycled
  the battery connector, not the AC side, which is very likely why they all came up empty.
  **The battery-present requirement makes sense structurally, not just as a firmware
  quirk: the EEPROM being read/written lives inside the battery pack, not on the charger
  board at all** (see "Physical access points" above) - an AC power-cycle with no battery
  attached has no EEPROM on the bus to talk to, full stop. Reliable repro recipe going
  forward: **battery connected, then power-cycle the charger's AC.**
- The sequence is a clean, complete "self-test" picture: a **27-address read sweep**,
  immediately followed by **writes into the `0x80`-`0xDF` log block**, then a **verify-read
  and mirror-write on `0x40`/`0x44`**, ending with a **live `+1` increment written to
  `0x50`**.

**Read sweep** (write-pointer + repeated-start + read-3-bytes, XOR-valid throughout) walked
these addresses in this exact order: `0x10, 0x20, 0x2C, 0x48, 0x4C, 0x30, 0x38, 0x40, 0x50,
0x60, 0x28, 0x24, 0x80, 0x84, 0x88, 0x8C, 0x90, 0x94, 0x98, 0x9C, 0x64, 0x18, 0x3C, 0x58,
0x5C, 0x08, 0x0C`. This list is itself useful independent of the values - it's the
charger's own idea of which registers matter at self-test/connect time, and it touches
several previously-`UNKNOWN`/`CONSTANT` ranges (`0x08`-`0x0F`, `0x64`-`0x73`, `0x3C`,
`0x58`-`0x5C`) that were assumed unused - worth re-flagging those as "probably meaningful,
just not yet decoded" rather than padding.

Decoded values from this sweep (all XOR-valid): `0x10`=299, `0x24`=500 (LE16+XOR - raw units
map 1:1 to Wh, matching Babboe's R50/500Wh model naming exactly - see revised note on the
`0x24` row above), `0x28`=13345 (BE16+XOR - matches the already-known constant exactly),
`0x30`=`0x40`=7456 (mirror match, consistent with the existing calibration-ceiling model),
`0x50`=1511, `0x18`=`07 1A 1D` (matches the known identity constant).

**Write burst into `0x80`-`0xDF`:** 8 writes at pointers `0xA0, 0xA4, 0xA8, 0xAC, 0xB0, 0xB4,
0xB8, 0xBC`, the last landing `50 50 00` - the exact terminator byte pattern already seen
in static dumps of this block. First direct proof this block is actively written by the
charger, not a static factory record (see updated `0x80`-`0xDF` row above).

**`0x40`/`0x44` refresh (~0.9s later):** wrote `20 1D 3D` (=7456, unchanged) to `0x40`,
immediately **read it back to verify**, then wrote the identical value to the `0x44` mirror.
Confirms both the mirror relationship and that this firmware does write-then-verify.

**`0x50` live increment:** the final transaction writes `E8 05 ED` to `0x50` - decodes to
1512, i.e. **exactly +1** from the 1511 read earlier in the same sequence. This is a
bracketed, single-event, caught-in-the-act increment - the strongest possible confirmation
for `session_counter`, and the reason `0x50`'s status above is now `CONFIRMED` rather than
`HYPOTHESIS`.

**Second capture, same day:** a repeat AC power-cycle produced an identical-shape sequence
(same 27-address read order, same `0x80`-`0xDF` write burst ending `50 50 00`, same
`0x40`/`0x44` verify-and-mirror), and `0x50` continued exactly where the first capture left
off: read `1512`, wrote `1513`. Two independent, sequential `+1` increments on two separate
power-cycles - about as strong as confirmation gets without a firmware source dump.

**Cross-tool confirmation:** after fixing two bugs in `Charger_LiveSniff.ino` (it was only
being tested with the wrong trigger - battery plug/unplug instead of an AC power-cycle -
and separately had a stack-overflow bug in its flash-rotation code causing hard crashes),
the ESP32 passive sniffer caught this same self-test sequence live and decoded it
byte-for-byte identically to the logic-analyzer captures above - same `0xA0`-`0xBE` write
burst values, same terminator, and `session_counter` continuing its unbroken `+1`-per-event
streak (`1522` -> `1523` across two more captures). Two independent tools, same result -
the decode pipeline itself is now trusted. The ESP32 capture also confirms this routine
self-test does **not** touch `0x1C`, `cal_stat1-3`, or the `0xE0`-`0xFF` cell tables -
only `0x40`/`0x44` (refreshed, unchanged value) and `0x50` (incremented) - reinforcing that
those other fields really are calibration-only, not part of every connect/self-test cycle.

## Confirmed 2026-08-14: full calibration cycle caught live, atomic write group finally seen directly

After leaving `Charger_LiveSniff.ino` running passively for an extended period (~28 hours
between checks) with a battery connected, a real calibration cycle completed on its own -
almost certainly the same one flagged as "Cal in progress" earlier - and the sniffer caught
the entire atomic write group in one shot, byte-for-byte, resolving several fields that
static dumps alone couldn't:

- **`0x30`/`0x34` (cal_measured) changed for the first time ever observed**: `7456` ->
  `6666`. Every dump before this had it frozen. Now `CONFIRMED`.
- **`0x40`/`0x44` jumped to exactly match the new `0x30` value** (`6666`) - the same
  "syncs to the calibration ceiling" behavior documented back in the very first calibration
  finding, now confirmed a second time with the ceiling itself moving.
- **`cal_stat1`/`2`/`3` populated with traceable source bytes**: `cal_stat1` (`0x74`-`0x76`
  = `1B 19 02`) = 6427 - identical to the value seen in a 2026-08-11 static dump, suggesting
  it's stable/repeatable rather than randomly re-generated each calibration. `cal_stat3`
  (`0x7C`-`0x7E` = `98 98 00`) = 39064 - close to but not identical to the earlier dump's
  39835. `cal_stat2` (`0x78`-`0x7A`) = 16448 this time vs 13621 in the earlier dump - the
  one of the three that clearly changes between calibration events, making it the best
  candidate for something calibration-run-specific (a fresh measurement) rather than a
  fixed status/config code.
- **A real cell-tap voltage write caught live**: `cell_bottom[3]` (`0xFC`-`0xFE`) =
  3152mV/3163mV, matching the range already seen in static dumps.
- **Two brand-new addresses discovered**: `0x20` (`00 00 00`) and `0x2C` (`02 00 02`,
  decodes to 2) - neither was in either decoder before now. Both sit in previously-untracked
  gaps between existing field groups and were written in the same atomic burst.
- **`0x10` incremented by exactly `+1`** (`364` -> `365`) in the same write group - the
  first time this field's movement has been tied to a specific, identified event
  (calibration completion) rather than an untraceable delta between two dumps.

This is the clearest single piece of evidence gathered so far for the "calibration writes
several fields as one atomic group" model first proposed back on 2026-08-11 - it's no longer
an inference from before/after diffs, it's a live transaction capture. It also reopens a
question about `0x40`'s day-to-day behavior: between calibration events, `0x40` was observed
decrementing by an extremely regular, fixed `-74` roughly every ~755 seconds regardless of
whether the battery was under real load (see the immediately preceding capture series) -
behavior far too mechanically regular to be a raw analog charge/discharge reading. The
working model going forward: `0x40`/`0x44` is a countdown-style gauge that decays at a fixed
rate between calibrations and gets reset to a freshly-measured ceiling (`0x30`) whenever a
calibration cycle completes - not a direct real-time current-integration reading.

## Confirmed 2026-09-06: second physical pack (Bat:003) - unused-pack state, serial number decoded

`GWA_Battery_Diag.ino`'s single 256-byte block read (`readEEPROM()`) failed completely
against this second pack (`EEPROM found at 0x50` on the address probe, then every full-block
read failed) while the exact same code worked fine on pack #1. Root cause was the long
single-transaction read itself, not the wiring or a different chip - falling back to
16-byte chunked reads (separate pointer-set + repeated-start per chunk) got a clean full
dump. This says the second pack's physical connection tolerates short transactions but not
one long 256-byte one; `readEEPROM()` now tries the single-block read first and automatically
falls back to chunked reads (with per-chunk error logging) if it comes up short.

The resulting dump revealed this pack has **never been through a live field calibration**:
`stored_capacity` (`0x1C`), `cal_stat1-3` (`0x74`-`0x7C`), and the entire cell-voltage table
(`0xE0`-`0xFF`) are all still factory-default `0xFF`. That "mostly blank" state turned out to
be useful - it isolates which fields *do* get set before any real calibration ever happens:

- **`0x24` rated_capacity reads `500` again** - identical to pack #1, despite this being a
  different physical unit. **Settled:** this pack's label reads `R37 / 11.4Ah`, confirming
  R37 and ruling out R50 - but since 11.4Ah doesn't convert cleanly from `500` under any
  tidy factor, `500` is now understood as a fixed R37-tier class code, not a literal Wh/Ah
  value (see the `0x24` row above and `[[project-cop2-capacity-units]]`).
- **`0x28` cal_cycles reads the exact same `13345`** as pack #1 - confirms this is a
  model-wide constant, not a per-pack or live value (row upgraded to `CONFIRMED`).
- **`0x30`/`0x40`/`0x44` all read `3936`** (XOR-valid) despite no live calibration ever
  having populated `0x34`/`cal_stat1-3`/cell tables. This means `0x30` can carry a
  **factory-set initial measurement**, separate from the "5 fields written atomically" event
  that only happens at a real *field* calibration - revising the `0x30`/`0x34` model (see
  those rows above).
- **`session_counter` (`0x50`) already reads `898`** on a pack that's never been calibrated
  or had a capacity level recorded - consistent with it counting every self-test/connect
  event (including factory QC, shelf/demo handling, etc.), not real charge/discharge use.
- **Previously totally-blank/unknown ranges turned out to be populated on this pack**:
  `0x14` (520), `0x3C` (10300), and all of `0x64`-`0x73` (four evenly-stepped values: 6000,
  6968, 7936, 8996) - likely more factory-set reference data, given this pack's otherwise
  uncalibrated state.
- **`0x18` identity reads `07 03 04`**, vs pack #1's `07 1A 1D` - different trailing bytes,
  same leading byte. Best read: leading byte = shared model constant, trailing 2 = a
  per-unit serial-like value.
- **Log block group 1 (`0x80`-`0x9F`) decodes to `BBAPTN5203039`**, and the user
  independently confirmed this exact string is printed as this pack's serial number on its
  physical label - directly solving what group 1 of the log block actually is (see that row
  above). Groups 2/3 use the same ASCII+XOR+`P`-pad+`PP`-terminator encoding but don't match
  the serial - still unidentified.

## Cross-referenced against public sources 2026-09-07

Searched for any existing GWA ibo-COP2 documentation, BMS chip datasheet, or third-party
reverse-engineering write-up. **None found** - no public EEPROM/I2C protocol docs, no
identified BMS/mux chip datasheet, no forum or GitHub write-up covering this specific
system. This register map appears to be the most detailed public source on this EEPROM
layout that currently exists.

What *did* turn up, from GWA Energy's own site (gwaenergy.com/Systems, packageid=13,
"ibo-09S eDrive System" - an official first-party source, not a reverse-engineered one):

- **Official Wh ratings: R37 = 374Wh, R45 = 447Wh, R50 = 499.5Wh.** Confirms `374` (already
  used in `GWA_Battery_Diag.ino`'s display formula) is GWA's real number for R37, and
  replaces the previously-guessed "~450/460" for R45 with a real figure (447). `447`
  doesn't scale cleanly from raw `0x24`'s `600` either - same non-1:1 pattern as R37,
  reinforcing that raw `0x24` is a tier code, not a display-formula input (see `0x24` row).
- **"499.5 Wh, 33.3V, 15Ah"** for the R50-tier pack: `499.5 / 33.3 = 15.0Ah` exactly (self-
  consistent), and `33.3 / 9 = 3.7V/cell` - a second, independent official confirmation of
  the 9S topology, alongside the charger datasheet already cited above.
- **"ibo-COP2 gives end-users a cell balancing function"** - officially confirms the COP2
  charger itself actively balances cells during charging (not just charges). Relevant to
  the parked balancing-feature idea in `STANDALONE_DEVICE_DESIGN.md`: a rescue-balance tool
  would be working alongside/instead of a charger that already does this, reinforcing why
  that design runs at rest (pack disconnected from its charger), not during a charge cycle.

See `[[project-cop2-capacity-units]]` and `[[project-cop2-pack-topology]]` in memory for
the full notes.

## Findings 2026-10-04: new scans from the GitHub scan database

Two new BLE scans uploaded on 2026-10-04: Bat:001 (`BAPDMP24R1264`, compared with its
2026-08-13 dump) and Bat:003 (`BBAPTN5203039`, after a Recode with retier 500 -> 600 and one
short charge test, compared with its 2026-09-29 scan). Field-level details are in the rows above.

- **Recode + short charge on Bat:003:** Recode wrote `0x24`=600, `0x1C`=600 (BE), `0x10`=0 and
  `0x50`=0. The charge then moved `0x50` to 1 (the usual +1 per power-on) and rewrote group 2
  (`0xA0`-`0xBF`) with the charger's ID. Nothing else changed: no calibration happened, so
  `0x2C`, `0x34`, `cal_stat1-3` and both cell tables are still blank. Note: Bat:003's label
  says R37 / 11.4Ah, but it is now coded as the R45 tier (600).
- **Bat:001 since 2026-08-13:** `0x50` +97, `0x10` +74, `0x60` +7, `0x2C` 1 -> 2,
  `0x1C` 433 -> 434. `0x30`/`0x34`/`0x40`/`0x44` all read 6666. The calibration fields and cell
  tables match the 2026-08-14 live capture exactly, so nothing has been recalibrated since then.
- **Bat:001 cell balance (from the 2026-08-14 calibration):** top-of-charge spread 15 -> 46 mV,
  bottom 8 -> 13 mV. Cells 1, 3, 6 and 7 (and possible cell 9) read low at both ends, which
  points to a state-of-charge imbalance rather than lost capacity. Not a concern yet; watch the trend.
- **Tool bug fixed:** `GWA_Battery_Diag.ino` counted blank `FF FF FF` records as XOR errors,
  which gave every uncalibrated pack a false EC#2 risk (30 on Bat:003). Blank records are no
  longer counted (`isBlankRecord()`).

## What's actually "locked down" vs what still needs a live probe

Static analysis of dumps (diffing, XOR-checking, cross-referencing against known values
like rated capacity) has taken every field as far as it can go for now. Genuinely
`CONFIRMED` status - meaning: encoding is right AND the real-world meaning is verified -
currently applies to **`0x24` rated_capacity**, **`0x1C` stored_capacity (BE16+XOR)**,
**`0x50` session_counter**, **`0x30` cal_measured**, **`0x28` cal_cycles (model-wide
constant, matched identically across 2 physical packs)**, and **`0x80`-`0x9F` (group 1 of
the log block = the pack's printed serial number, matched directly against the physical
label)** (see the 2026-08-13, 2026-08-14, and 2026-09-06 sections). Everything else below is
`HYPOTHESIS` with varying degrees of supporting evidence, and closing them out requires the
specific physical actions already defined in `PROBE_PROTOCOL.md` - no further staring at
existing dumps will move them further:

| Field | What's missing | Probe action needed |
|---|---|---|
| `0x10` plug-ins/charge_events | Refuted 2026-10-04: it also moves outside calibration (+74 vs one calibration). Still open: exactly which connects increment it (+74 vs +97 on `0x50`) | compare `0x10` and `0x50` before and after single charges, full vs partial |
| `0x40`/`0x44` totalizer | Confirm the fixed `-74`/~755s decay rate is real and find the units (mAh vs Wh vs raw); confirm it always resets to `0x30` at calibration | `charge_timed` with known bench current, or just keep passively logging |
| `0x34` mirror (reopened) | Whether it only gets written by a *live field* calibration (unlike `0x30`, which can be factory-preset - see 2026-09-06 finding) | catch a live calibration on a pack that starts with `0x34` blank, confirm it populates |
| `0x20`/`0x2C` | `0x2C` is very likely the calibration count (2026-10-04, all 3 packs agree); `0x20` still unexplored | scan Bat:003 after its first full CAL cycle: `0x2C` should read 1 |
| `cal_stat2` (`0x78`) | Why it changes between calibrations while `cal_stat1`/`3` repeat identical values | compare across 2+ more calibration events |
| `0xE0`-`0xFF` cell taps | Pack is confirmed 9S, so 16 raw values can't all be independent series cells - figure out which of the 16 are real per-cell taps vs. redundant/dual-bank/unused channels, and confirm the `+4000`/`+3000` offsets | `multimeter_taps` against all 9 physical cells |
| `0x80`-`0xDF` groups 2/3 | Group 2 very likely = last charger's ID (2026-10-04); group 3 is never touched by the charger | charge a pack on a different charger and check that group 2 changes; check whether the bike's battery dock has data contacts |
| `0x00`-`0x02` status/stage | Bit-level meaning; whether `0x01`/`0x02` are transient stage markers or persist | `fault_trigger`, watch across more self-tests |
| `0x18` identity | Leading byte looks shared (model constant), trailing 2 bytes differ per pack (serial-like) - want a 3rd pack to be sure | dump a 3rd physical pack, compare |
| `0x64`-`0x73` (new) | Fully populated, evenly-stepped 4-value table found 2026-09-06 - meaning totally open | compare against a second populated pack |
| `0x60` cap_threshold | Whether it's really "~71% of rated" (358 on pack #1) or something else entirely (155 on pack #2, which has never been calibrated) | compare on a freshly-calibrated pack |
| `0x24` rated_capacity | Class codes confirmed for both tiers (`500`=R37, `600`=R45 - not GWA's own `596` constant), but the actual real-Wh conversion GWA displays is unverified - is `374/500` accurate for *any* real R37, or just a rough approximation? | weigh/bench-test a pack at known SoC and compare against the displayed Wh |
| `0x14` / `0x3C` | Populated on Bat:003 and the R45 pack, blank on Bat:001 - doesn't split by model or by pack, likely tracks firmware/calibration-history version | compare against a freshly-calibrated Bat:001 dump, or a pack of known firmware revision |
| `0x0C`-`0x0E` (new) | Populated only on the R45 pack so far - genuinely model-specific, or just this one unit? | dump a second R45 pack |

Run these through `tools/probe_diff.py` as you go (see `PROBE_PROTOCOL.md`) so each
result accumulates in `hypothesis_ledger.json` instead of needing to be re-derived by
eyeballing dumps each time.

See `PROBE_PROTOCOL.md` for how to run these as isolated, repeatable experiments and
`tools/probe_diff.py` for turning the resulting dumps into confidence-scored evidence
instead of one-off eyeballing.
