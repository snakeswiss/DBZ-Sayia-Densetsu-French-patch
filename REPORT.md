# DBZ Super Saiya Densetsu (SNES) — FR translation: Frieza final-form freeze — Root Cause & Fix

**Date:** 2026-07-06
**Input:** `Dragon_Ball_Z_-_Super_Saiyan_Densetsu_Translated_FR.smc` (1,573,376 B = 512 B copier header + 1.5 MB expanded LoROM)
**References:** JP original (1 MB), Klepto EN v1.02 (1 MB) — both verified working baselines.
**Symptom:** Hard freeze at the end-game when Frieza transforms into his 4th (final) form. JP and EN complete normally.

---

## 1. Summary

The freeze is caused by **corrupted entries in the battle-unit NAME table** of the FR ROM (bank `$82`). Three independent data errors — all introduced by the FR translation, all in the same table — make the name-rendering routine read absurd string lengths (37 and 100 bytes) through mid-string pointers. When the battle engine swaps Frieza to his final-form unit and draws its name, it copies ~100 bytes of arbitrary data (including bytes the text engine treats as control codes) into a small name buffer → engine lockup.

The user's hypothesis ("text overflow corrupting something essential, probably in the end-game enemy names") was exactly right.

Total fix: **12 functional bytes + 4 header-checksum bytes.**

## 2. Text system architecture (relevant parts)

* FR is based on the **EN (Klepto v1.02)** ROM (54 KB diff vs EN, 191 KB vs JP), expanded 1 MB → 1.5 MB. The FR team added a custom dialogue engine in the expansion (bank `$20`, routine at PC `0x100000`, hooked from bank `$80` via `JSL $208080`-style calls). Dialogue banks `$86`/`$87` therefore use an FR-specific encoding — untouched by this fix.
* **Unit names** do NOT go through that custom engine. They use the stock reader:
  * Pointer table **table2** at PC `0x12CE8` (SNES `$82:ACE8`) — **105 × u16 in-bank pointers**, one per battle unit (players + enemies).
  * Each pointer targets a **length-prefixed string**: `[len:u8][chars…]`.
  * Reader code (identical in JP/EN/FR), e.g. at PC `0x0BC4C`‑area (SNES `$81:BC4C`):
    `LDA $0E1C,Y` (unit idx) → `ASL` → `LDA #$ACE8 / STA $00` → `LDA [$00],Y` (fetch ptr) → length-prefixed copy via `JSL $01BC22`.
  * Adjacent tables: table1 `0x12B66` (44 ptrs, menu/system), table3 `0x12EA6` (69 ptrs, technique names).
* In EN, **all 8 Frieza-form units share one string**: ptr `AE7A` → `06 "Frieza"`. FR renamed it in place to `07 "Freezer"` (used the padding byte — fine).

## 3. Root causes (3 defects, 13 broken table entries)

### Defect A — "Yajiro" overflow clobbers a length byte  *(breaks entries 85–95)*
FR edited the inline name at `$82:AE95` from `05 "Yajir"` to **"Yajiro" (6 chars) without room**: the final `o` (`0x25`) was written at PC `0x12E9B`, which is the **length byte (`0x08`) of the next string** — an 8-byte unit name shared by **11 consecutive units** (entries 85–95, the story/Ōzaru-type units). Their length became `0x25 = 37`, so any name fetch for those units reads 37 bytes, spilling across the remaining names **into pointer table3**.

### Defect B — entry 100 points mid-string  *(Frieza form unit)*
EN: `t2[100] = AE7A ("Frieza")`. FR: `t2[100] = AE84` → lands **inside "Gokou"** (`AE82`). Byte at `AE84` = `0x25` → phantom length **37**.

### Defect C — entry 104 points mid-string  *(Frieza FINAL form — the reported freeze)*
EN: `t2[104] = AE7A`. FR: `t2[104] = AE89` → lands **inside "Dendé"** (`AE88`). Byte at `AE89` = `0x64` → phantom length **100**. Entry 104 is the last unit in the table (final/100% form). The 100-byte "name" sweeps through every remaining string and the whole technique pointer table; the copied garbage includes `0xF5/0xAF/0xB4/…` bytes that the renderer interprets as control codes → **freeze at the 4th-form transformation**, exactly as reported.

Full FR validation of tables 1/2/3 found **no other defects** — the 13 bad entries above all reduce to these 3 root causes. Banks `$86`/`$87` (dialogue) pointer sets are structurally sound; the `$88`/`$8A` diff regions are graphics (redrawn font/tilemaps), not text tables.

## 4. Fix (applied)

| # | PC offset | .smc offset (+0x200) | SNES | Old | New | Effect |
|---|-----------|----------------------|------|-----|-----|--------|
| 1 | `0x12DB0` | `0x12FB0` | `$82:ADB0` | `84 AE` | `7A AE` | t2[100] → "Freezer" |
| 2 | `0x12DB8` | `0x12FB8` | `$82:ADB8` | `89 AE` | `7A AE` | t2[104] (final form) → "Freezer" |
| 3 | `0x12E9B` | `0x1309B` | `$82:AE9B` | `25` | `08` | restore shared-name length (entries 85–95) |
| 4 | `0x175B6` | `0x177B6` | `$82:F5B6` | `FF ×7` | `06 45 05 1E 22 29 25` | new string `"Yajiro"` in free space |
| 5 | `0x12D8E` | `0x12F8E` | `$82:AD8E` | `95 AE` | `B6 F5` | t2[83] Yajirobé → relocated "Yajiro" (translator's intended 6-letter name, now without overflow) |
| 6 | `0x07FDC` | `0x081DC` | header | `02 CC FD 33` | `5A 33 A5 CC` | internal checksum recomputed (standard 1 MB + 2×0.5 MB mirror rule); cosmetic only |

FR letter codes used in "Yajiro": `Y=45 a=05 j=1E i=22 r=29 o=25` (taken verbatim from the translator's own bytes at `AE95`).

## 5. Verification performed

* Byte-level audit: exactly **16 bytes** differ between original FR and fixed FR (listed above) — nothing else touched.
* Post-fix validation of all 105 table2 entries: **0 anomalies**; all 8 Frieza-form entries resolve to `07 "Freezer"`; entries 85–95 resolve to the original 8-byte string (byte-identical to EN); t2[83] resolves to `06 "Yajiro"`.
* `DBZ_SSD_FR_freeze_fix.ips` applied to the original uploaded `.smc` reproduces the fixed ROM **byte-for-byte** (round-trip tested). IPS offsets are for the **headered** file (512-byte header included).
* Fixed ROM MD5: `39481d86e43f4ba308fc1c25c1952a32`.

## 6. In-game test checklist

1. Final battle: Frieza forms 1→2→3→**4** (the previously freezing transition) and any 100%/Super Saiyan phase, through to the ending + credits.
2. Ginyu fight incl. Body Change (name-swap heavy).
3. Any story/Ōzaru special battles (units 85–95 — their name length was silently broken too; they may have displayed garbage or over-long name boxes before).
4. Yajirobé screens — should now display "Yajiro".

*Static analysis note:* the dialogue banks use the FR team's custom engine (bank `$20`); they validated structurally but only in-game play proves them. Everything on the crash path itself (name table + stock reader) is now provably consistent with the working EN baseline.
