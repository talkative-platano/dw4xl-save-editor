# DW4 XL (SLUS-20812) save format — reverse-engineering notes

> **Active tool = `dw4xl_save_editor.html`** (single-file Vue GUI: load .psu, edit
> player/char stats, bodyguards, items, equipped gear; pending-changes list; reset;
> download with recomputed checksum). All offsets/roster/item-names below are baked
> into its consts and validated lossless in Node. `dw4edit.py` is legacy; the
> `.py`/`diff.py`/`psu.py` scripts remain handy for ad-hoc RE/diffing only.

## Container
`*.psu` = PCSX2/mymc memory-card export. Inside it the real game save is
**`BASLUS-20812`, 34064 bytes**, located at offset `0x1c000` in the PSU.
(`.sys` = save metadata, `u.ico` = 3D icon — irrelevant.)
Use `psu.py` to list/extract, the HTML editor (or `dw4edit.py`) to read/write fields.

## Checksum (CRITICAL)
`u16 little-endian @ offset 0x00  ==  sum(save[4:]) & 0xFFFF`
Bytes 2–3 are a constant `0x0003` (version), excluded from the sum.
**Recompute after any edit ≥ offset 4** or the save is rejected.
Proven: edited clean→easy by hand and reproduced the real easy save byte-for-byte.

## Confirmed fields
| What | Offset | Type | Notes |
|---|---|---|---|
| Checksum | `0x00` | u16le | sum(save[4:]) |
| Difficulty | `0x9a` | u8 | 0=Novice, 1=Easy, 2=Normal (default), 3=Hard, 4=Expert |
| Points (per char) | char record `+16` | u16le | matches in-game exactly (Zhao 13882, Guan 5717) |
| Weapon EXP | `0x798 + 2*idx` | u16le | per-character array; Zhao=2366, Guan=310 ✓. Weapon level is derived from this, not stored (see below) |
| Bodyguard team points | team `+94` | u16le | per-team; ROTK=150 ✓ (see below) |

Roster index = in-game character order, **42 characters total**:

```
 0 Zhao Yun     1 Guan Yu      2 Zhang Fei    3 Xiahou Dun   4 Dian Wei
 5 Xu Zhu       6 Zhou Yu      7 Lu Xun       8 Taishi Ci    9 Diao Chan
10 Zhuge Liang 11 Cao Cao     12 Lu Bu       13 Sun Shang Xiang 14 Liu Bei
15 Sun Jian    16 Sun Quan    17 Dong Zhuo   18 Yuan Shao    19 Ma Chao
20 Huang Zhong 21 Xiahou Yuan 22 Zhang Liao  23 Sima Yi      24 Lu Meng
25 Gan Ning    26 Jiang Wei   27 Zhang Jiao  28 Xu Huang     29 Zhang He
30 Zhen Ji     31 Huang Gai   32 Sun Ce      33 Wei Yan      34 Pang Tong
35 Meng Huo    36 Zhu Rong    37 Da Qiao     38 Xiao Qiao    39 Cao Ren
40 Zhou Tai    41 Yue Ying
```
Editor groups the dropdown by faction (Wei/Shu/Wu/Unaligned) via `FACTION_OF`.

## Weapon EXP & Lv.10/Lv.11 weapon
Weapon level is **purely EXP-derived** (`0x798 + 2*idx`, u16le) — there is no separate
weapon-level or unlock byte.
- Normal play caps weapon EXP at **36000** (covers weapon levels 1–10).
- **36001** = the special **"Lv.10" weapon**, **36002** = the **"Lv.11" weapon**.
- The in-game "special condition" only gates *earning* those final point(s); there is
  NO separate unlock flag — set the EXP and you have the weapon.

Editor: Weapon EXP is its own numeric row (0–36000, clamped unless "no input validation"
is on) plus "Lv.10"/"Lv.11" checkboxes that force 36001/36002.

## Bodyguard teams
Array of **4 teams**, 96-byte records starting at **`0x508`** (stride 96).
Order: 0=ROTK, 1=Koei, 2=Mystery, 3=Kessen.

| Field | Rec offset | Notes |
|---|---|---|
| Team name | `+0` | 9-byte ASCII, null-terminated (max 8 chars) |
| Guard names | `+9 + 9*k`, k=0..7 | 8 guards, 9 bytes each |
| Team stats | `+82..+90` | `8c 8c 2d 3c 14 14 6e 50 78` — identical for all teams |
| Points | `+94` | u16le |

Each team has up to 8 guards (2 unlocked initially). **Points (`+94`) is the only stored
per-team value**; team stats and unlocked-guard count are derived from points (the stat
bytes are identical across teams and didn't change when ROTK reached 150). Guard/team
names are preset Koei references and are editable strings.

## Character stat table
Array of **24-byte records** starting at **`0xb8`**, **42 records** (one per roster char;
table ends cleanly at idx 42). Record 0 = Zhao Yun.

| Field | Rec offset | Zhao @ | Meaning | Evidence |
|---|---|---|---|---|
| present | `+0` | 0xb8 | `0x01` marker | constant |
| Life | `+1` | 0xb9 | u8, base 145 | **confirmed**: Cao Cao (idx 11) Life-up moved this byte 140→150 |
| Musou | `+2` | 0xba | u8, base 165 | **confirmed** |
| Attack | `+3` | 0xbb | u8, base 50 | **confirmed**: Atk+Def=100, single-stat test |
| Defense | `+4` | 0xbc | u8, base 50 | **confirmed**: Guan Yu +2 def moved only `+4` (46→48) |
| char index | `+5` | 0xbd | 0,1,2,… | increments per record |
| class | `+6` | 0xbe | 3/4/5 | group flag |
| unk7 | `+7` | 0xbf | u8, base 46 | DEcreases with play (purpose TBD) |
| **Equipped items** | `+8..+15` | 0xc0 | 8 item-table indices | see below |
| Points | `+16..+17` | 0xc8 | u16le (maybe u32) | accumulated points |
| pad | `+18..+23` | 0xca | zero | |

All four are confirmed individually: `+1`=Life, `+2`=Musou, `+3`=Attack, `+4`=Defense.

### Equipped items (`+8..+15`)
8 bytes, each an **index into the item table** (same order as the item-name list below);
`0x29` (=41, one past the last item) = **empty slot**. This is NOT a "caps" field — the
all-`0x29` pattern seen earlier was just empty slots on an un-equipped character.
- Slot 0 = harness, slot 1 = orb, slots 2..7 = general items (usable count grows with weapon EXP).
- Verified: Zhao's lv10 block `16 0f 0c 04 1d 29 29 29` =
  Shadow Harness / Vorpal Orb / Herbal Remedy / Speed Scroll / Wind Scroll / empty×3.
- Editor exposes these as 8 dropdowns ("Equipped").

## Items (global, not per-character)
Byte array of **41 items** at **`0x7f6`** (one byte each).
- `0xff` = locked / not found
- otherwise **level = byte + 1** (so found-at-Lv1 = `0x00`)
- Leveled items (idx 0–12) cap at Lv20 (`0x13`); **orbs (idx 13–18) cap at Lv4** (`0x03`);
  bool items (idx 19–40) are just locked (`0xff`) vs unlocked (`0x00`).
- Items are fixed (one of each; no duplicates).

All 41 identities are known and baked into the HTML (`ITEM_NAMES`) — there is no longer
any names.json / localStorage:

```
 0 Peacock Urn       1 Dragon Amulet    2 Tiger Amulet     3 Tortoise Amulet
 4 Speed Scroll      5 Wing Boots       6 Huang's Bow      7 Nanman Armor
 8 Horned Helm       9 Cavalry Armor   10 Seven Star Sash 11 Elixir
12 Herbal Remedy    13 Fire Orb        14 Lightning Orb   15 Vorpal Orb
16 Ice Orb          17 Blast Orb       18 Poison Orb      19 Red Hare Harness
20 Hex Mark Harness 21 Storm Harness   22 Shadow Harness  23 Elephant Harness
24 Art of War       25 Survival Guide  26 Bodyguard Manual 27 Way of Musou
28 Power Scroll     29 Wind Scroll     30 Fire Arrows     31 Charge Bracer
32 Power Rune       33 Code of Chivalry 34 Meat Bun Sack  35 Musou Armor
36 Master of Musou  37 War Drum        38 Secret of Orbs  39 Helm of Might
40 Horseshoes
```
Confirmed by edit: item idx 14 = **Lightning Orb** (`0x804`), found-at-Lv1 → `0x00`.
(Separate 17-byte `0xff` block at `0x1968` is a different unlock table tied to the
secondary character table near `0x19b0` — not the item table.)

## Not stored in the save (don't look for them)
- **Weapon level** — derived from weapon EXP (`0x798+2*idx`); see the Weapon EXP section.
- **Battle rank (16→1)** — a per-mission result. The `0x0d`/`0x09` bytes that look
  like rank are constants present in the clean save; nothing transitions to the
  observed rank. Not persisted. (Confirmed across Zhao=9 and Guan=13.)

## Other tables still to map (change with play, meaning TBD)
- `0x1980 + idx` : per-character byte, default `0x04`, becomes `0x02` once that
  character is played (Zhao & Guan both 2).
- `0x19ad` : global counter = number of characters played (1 → 2).
- `0x820`+ : array, default `0x06`, a few entries decrement with play.
- record `+7` (Zhao `0xbf`): per-char byte, base 46, drops ~7–8 each play session.
- Second character table near `0x19b0` (holds base/default stats; not the live one).
