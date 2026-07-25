# Handoff — Dome City appearance services (Cyber Cuts / Genolab)

Branch: `npc-edit-face-hair` (from `character-config-fix`)
Session: 2026-07-25

Goal: get the in-game hair / face-and-skin change vendors working, ideally
usable from the VR arena while Dome City isn't hosted.

---

## TL;DR of where we ended

The whole client-side chain is **intact and unstripped**. The client builds a
complete `UPDATE_CHAR_VISUAL_SETTINGS` (0x01D3) packet containing the chosen
hair mesh and every morph (node index, weight) pair, and hands it to the same
send path that all its other traffic uses. Our server-side handler for that
packet is written and waiting.

**It never gets sent.** Proven at the socket: with raw byte logging on the
control server's read path, clicking Apply produces *no bytes at all*. The
failure is inside the client, between the Apply button and `CompleteTransaction`.

Everything else on the path was broken and is now fixed (vendors, prices,
currency type, inbound framing). One client-side link remains — see
[Where to pick up](#where-to-pick-up).

---

## What the feature actually is

There is no special NPC class. The "barbershop" is a **UI volume** you stand in,
with a decorative `TgSkeletalMeshActorNPCVendor` beside it.

| Name | ui_volume_id | UI scene (res) | Loot table | Service item |
|---|---|---|---|---|
| **Cyber Cuts** (hair) | 140 | 6316 `GA_Menu_DomeCityCustomShops.Hair.GA_Menu_Hair_Mod2` | 495 | 6019 |
| **Genolab** (face/skin) | 177 | 6474 `GA_Menu_DomeCityCustomShops.GA_Menu_FacialReconstruct` | 496 | 6128 |

Client classes: `TgUIUpdateHairMenu`, `TgUIUpdateFaceMenu` (both extend
`TgUIVendorBase`). Both are fully present in the retail binary.

Flow: `U` in the volume → `TgPlayerController.ServerPerformedUseAction` →
`r_CurrentOmegaVolume.Used()` (runs on **both** sides; the client's copy opens
the scene) → menu → Submit → confirm popup → Apply → `CompleteTransaction`.

**Console shortcut (works today):** `TgPlayerController` has exec natives
`OpenUpdateHairScreen` / `OpenUpdateFaceScreen`. Bound during this session via
`tggame/Config/TgInput.ini`, appended at the end of `[TgGame.TgPlayerInput]`:

```ini
Bindings=(Name="F5",Command="OpenUpdateHairScreen",...)
Bindings=(Name="F6",Command="OpenUpdateFaceScreen",...)
```

UE3's `GetBind` scans backwards, so being last wins over the existing F5 binds.
**Verified equivalent to using the real volume** — same requests, same
behaviour, same failure. So the entry point is *not* a constraint, and the
feature does not need Dome City to be hosted.

---

## Fixes made this session (all in the commit)

### 1. Every vendor in the game was serving an empty inventory

`send_get_loot_table_items_by_id_filtered_response` was a hardcoded stub that
ignored the request and always returned loot table item 3470 / item 2991 (an
Agonizer) with a fabricated `AUTO_CREATE_FLAG` of 122319617.

Replacing it with real DB data wasn't enough — inventories were *still* empty.
The client's parser (hair menu @ `0x114bb786`, vendor scene @ `0x114b3385`)
reads exactly:

* items (`DATA_SET` 0x010C): `LOOT_TABLE_ITEM_ID` 0x572, `ITEM_ID` 0x2DA,
  `ITEM_SPECIAL_TYPE_VALUE_ID` 0x5DB
* prices (`DATA_SET_PRICES` 0x5F4): `LOOT_TABLE_ITEM_ID` 0x572,
  `CURRENT_PRICE` 0x5F6, `PRICE` 0x3D6, `CURRENCY_TYPE_VALUE_ID` 0x5F5

We were sending four fields it never reads, including `AUTO_CREATE_FLAG`
(0x005D), declared `TYPE_TCP_WCHAR_STR` but written as four raw bytes. Field
width comes from the field id, so that desynced the TLV reader mid-record and
lost the entire data set. **Send only what the parser reads.**

Result: PetCraft, Mercenary Armor, the dye/class/fashion vendors — all populate
now. This was never barber-specific.

### 2. Currency type matters, and 1603 is wrong

The response parser sets `m_bHasAPPrices` (bit 1 of `+0x1C8`) for currency
**1605** (Agenda Points) and `m_bHasNonAPPrices` (bit 2) for **1604** (Tokens).
It sets *nothing* for **1603** (Credits). With neither flag set, the menus treat
the item as unpriced and **disable Submit** (a disabled UI button also stops
responding to hover — that was the diagnostic tell).

Price-less items now get a synthesized row at price 0, currency 1604.

### 3. Inbound chunked messages were silently destroyed

`send_response` chunks payloads over 1456 bytes as
`[0x0000][1456 bytes]…[len][remainder]`, and the protocol is symmetric. But
`do_read` treated a zero length prefix as an *empty frame*: it dropped the
2-byte header and then read the chunk's payload as a new frame length. Garbage
length, desynced stream, message gone — and no "unknown packet type" log,
because no coherent frame is ever assembled.

Now reassembled properly (`rx_pending_`, `kChunkPayload` hoisted to class scope).
Latent for any large client→server message. **Not** the barber bug (the client
sends nothing at all), but a real defect found on the way.

### 4. `UPDATE_CHAR_VISUAL_SETTINGS` (0x01D3) handler

Written, ready, currently never invoked. Parses `CURRENCY_TYPE_VALUE_ID`,
`PRICE`, `LOOT_TABLE_ID`, `LOOT_TABLE_ITEM_ID`, `ITEM_ID`, `HAIR_ASM_ID` and
the `DWORDS` (0x222) morph blob, then persists via
`PlayerSessionStore::UpdateCharacterVisuals`.

The blob is the same layout `ADD_PLAYER_CHARACTER` sends — 4-byte length prefix,
then alternating `uint32` (morph node index, weight) — so it drops straight into
`ga_characters.morph_data`, which `CosmeticEquip::LoadFromDB` already scatters
into `r_nMorphSettings` at spawn. The face menu sends no `HAIR_ASM_ID`; 0 means
"leave alone".

### 5. Control server logs to files

It only ever wrote to stderr, so anything it saw had to be scraped from the
console. Now also appends to `<log_dir>/control-<channel>.txt`.

Note: the control server **never populates `EnabledChannels` from config**, so
all of its channels always log. The `enabled_channels` setting in
`control-server.json` only affects the game DLL.

---

## Findings that are NOT bugs (don't re-investigate)

* **`CanItemBePurchased` (`0x109d0fa0`) is a proximity check**, not an
  inventory check. It walks the pawn's `Touching` array (`AActor+0x100/0x104`)
  for an omega volume whose ui-volume record is type 1253 (Vendor) with a
  matching loot table id; loot table 480 is hardcoded exempt. **It passes** —
  proven by a successful armor purchase reaching the server.
* **`ServerPurchaseItem` reaches us correctly** from normal vendors:
  `lootTable=226 item=3933 count=1 currency=1604 cost=0`. Items don't appear
  because the grant is unimplemented — the native is a stripped stub and our
  hook just calls through. **This is the obvious next feature.**
* **The appearance services do NOT use `ServerPurchaseItem`.** They use the
  0x01D3 TCP packet instead. Nor do they use `ServerOnSetPlayerMesh`, the
  marshal control channel, or any script `Server*` RPC — all four were
  instrumented and came back empty.
* **UI volume definitions are client-local.** Nothing in our server ever sends
  `DATA_SET_UI_VOLUMES` (0x01C3); the game loads them from `tggame/Content/
  assembly.dat`, and `asm_data_set_ui_volumes` in `server.db` is only a capture
  mirror (`AsmDataCapture::WalkUiVolumes`). **Editing that table changes nothing
  in game.** Migration v153 tried to re-point VR volumes 126/237 at Cyber Cuts /
  Genolab on this false premise; **v154 reverts it**.
* **Dome City boots**: `home_map_name = "DomeCity_P_VER_3"`,
  `home_map_game_mode = "TgGame.TgGame_City"`. Cooked map files are in
  `tggame/CookedPC/Maps/CITIES/`. Useful as a lab; not hosted (1 core/instance,
  5 cores on the VPS).

---

## The client chain, reversed

Verified addresses (client base `0x10900000`).

`UTgUIUpdateHairMenu` vtable = **`0x11892380`** — authoritative, from the
constructor at `0x114bb8b0` (`mov dword ptr [esi], 0x11892380`).

> Beware: deriving the vtable base by matching a single function to a slot gives
> `0x11892284`, which is wrong by `0xFC` and makes half the slots resolve to
> shared base-class routines. That mistake led to a false "the natives are
> stripped" conclusion. Anchor on the constructor.

| slot | method | address |
|---|---|---|
| +0x11C | `PostOpenScene` | `0x114ba440` |
| +0x120 | `PreCloseScene` | `0x114bb4a0` |
| +0x130 | `TickTgUIScene` | `0x114b9470` |
| +0x1E0 | (base) close confirm popup | `0x113aee60` |
| +0x1EC | `OnConfirmYesClicked` | `0x114b9530` |
| +0x21C | `SubmitButtonPressed` | `0x114ba1d0` |
| +0x234 | `UpdateSliderDisableStatus` | `0x114b9e20` |
| +0x23C | `UpdateCreditsInfo` | `0x114bb140` |
| +0x240 | `CompleteTransaction` | `0x114bae70` |
| — | **the 0x01D3 sender** | **`0x114b9bb0`** |
| — | loot-table response parser | `0x114bb5d0` |
| — | face menu's 0x01D3 sender | `0x114b4691` (opcode write site) |

**`CompleteTransaction` (`0x114bae70`)** — reads the selected entry out of
`m_HairStyleMorphNode` (`+0x1DC`) for the hair mesh id, walks `m_HeadAnimTree`
(`+0x1D8`) collecting each morph node's index (`byte [node+0x99]`, ≤ 0xFE) and
weight (0–255) into parallel arrays, writes them into the paper doll at
`[m_PaperDoll + idx*4 + 0x7CC]`, then calls the sender with
`(hairMeshId, &pairs)`. Unconditional — no early return.

**The sender (`0x114b9bb0`)** — three silent guards, then a balance check, then
build + send:

```
cmp [this+0x180], 0        je out    ; m_PlayerPawn
mov ecx,[this+0x1F4]       je out    ; m_LootTableItem
test byte [this+0x1C8], 6  je out    ; m_bHasAPPrices | m_bHasNonAPPrices
...
price = lookup(currency); balance = [pawn+0x154C] r_nToken  (or +0x1550 r_nHZPoints if AP)
cmp price, balance;  jg  -> show msg 0xD363 "cannot afford", return
...
0x5F5 currency, 0x3D6 price, 0x319 lootTableId, 0x572 lootTableItemId,
0x2DA itemId (hardcoded 0x1783 = 6019), 0x282 hairAsmId, 0x222 morph bytes
mov word ptr [marshal+0x26], 0x1D3
mov ecx, 0x119a0150 ; call 0x10921180      ; <-- identical to the send used by
                                           ;     GET_LOOT_TABLE_ITEMS, which
                                           ;     reaches us every time
```

All four conditions were verified satisfied in game:

| guard | evidence it passes |
|---|---|
| `m_PlayerPawn` | the loot-table request reads `[pawn+0x6cc]` for the profile id and we receive `PROFILE_ID=679` |
| `m_LootTableItem` | `SubmitButtonPressed` checks it before opening the popup, and the popup opens |
| `flags & 6` | flipping our currency 1604→1605 switched the confirm text and balance line to "Agenda Points" |
| balance | price 1 vs a granted 1,000,000; the "cannot afford" message never appeared |

**`OnConfirmYesClicked` (`0x114b9530`)** is correctly wired:

```
call [vt+0x1E0]    ; close popup
call [vt+0x240]    ; CompleteTransaction
mov eax,1; ret 4
```

---

## Where to pick up

The contradiction to resolve: every guard demonstrably passes, the send path is
byte-identical to one that works, `OnConfirmYesClicked` calls
`CompleteTransaction`, the popup does close on Apply — and yet **zero bytes**
leave the client.

The most likely gap is that the popup's Apply button never reaches
`OnConfirmYesClicked` at all, and the popup closes via a generic base handler
instead. Supporting evidence:

* `SubmitButtonPressed` only sets an **action tag** `[this+0x15C] = 7` plus
  three labels (`0x104` body ← msg 0xD474 + price, `0x148` yes ← 0xD475,
  `0x150` no ← 0xD476). It binds **no** Yes callback.
* The base close routine at `0x113aee60` clears `0x15C` and two pairs at
  `0x164/0x168` and `0x170/0x174` — which look like Yes/No delegate slots that
  nothing ever filled.
* `TgUIUpdateHairMenu.FixupWidgetsUC` (UC) binds close/cancel/submit/sliders but
  **not** the confirm buttons — so the binding must be native.

**Next concrete step:** find the base scene-driver code that reads the action
tag at `+0x15C` and dispatches on it, and see what it does for value **7**.
Reader sites already located (`cmp dword [esi+0x15C], imm`): `0x113b2c74`,
`0x113b4f89`, `0x113f6940`, `0x1140594a`, `0x11405a06`, `0x1142cf69`, plus
`mov eax,[esi+0x15C]` at `0x113b4f64`, `0x113b5038`, `0x11405965`. Note
`0x113b4f64` is only a layout helper — it tests 2,3,4,5,6,8 and **not 7**.

Tooling: `scratchpad/gadis.py` (capstone disassembler over
`out/client/Binaries/GlobalAgenda.exe`, base `0x10900000`) with helpers
`dis(va)`, `refs(va)`, `rd4(va)`, `fn_extent(va)`, `o2v/v2o`. Ghidra MCP is
loaded with the same binary (functions unnamed; `list_strings` + `get_xrefs_to`
are the useful entry points). Exec-thunk → vtable-slot technique: find the
registration string (e.g. `intUTgUIUpdateHairMenuexecCompleteTransaction`), the
data xref to it, then `+4` is the thunk, whose final `mov edx,[eax+SLOT]; call
edx` gives the slot.

**Client-side instrumentation is not available** — injecting the client DLL
(`out/client/version.dll`) makes the game close immediately on this setup, so
this has to be settled by static reading or by server-observable side effects.

### If that dead-ends

* **Implement `ServerPurchaseItem`** — unrelated to the barber but it's the
  reason nothing you buy anywhere ever arrives, and it's fully server-side.
* **Character-creation path already works** end to end and sends the identical
  morph blob (that's how `morph_data` gets populated today). A "recustomize"
  flow driven from character select is a viable fallback for the same result.

---

## Test recipe

1. `home_map_name = "DomeCity_P_VER_3"`, `home_map_game_mode = "TgGame.TgGame_City"`
   in `control-server.json` (or stay in VR and use F5/F6 — verified equivalent).
2. Rebuild control server (+ DLL if game-side changes).
3. Walk to Cyber Cuts / Genolab, or press F5 / F6.
4. Move a slider → Submit → Apply.
5. Watch `out/logs/control-tcp.txt`. Success looks like:
   `UPDATE_CHAR_VISUAL_SETTINGS: charId=… lootTable=495 item=6019 … morphBytes=…`
   followed by `persisted=1`, then relog to see it applied.

Temporary aids used during the session, removed from the commit — re-add if
needed: raw socket hex logging in `do_read`; a `barber`-channel log in the
`ServerPurchaseItem` / `ServerOnSetPlayerMesh` hooks; a `Server*` filter in
`UObject__ProcessEvent`; a currency grant
(`r_nCurrency`/`r_nToken`/`r_nHZPoints`, all `CPF_Net` and already in the V2 rep
list, so a server-side write is the whole mechanism) in `SpawnPlayerCharacter`.
