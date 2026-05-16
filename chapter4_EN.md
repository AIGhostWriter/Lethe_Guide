# Chapter 4 — Full Sample: Soji Abi (소지아비)

> A complete boss mod built from scratch. Every file shown is the actual file used.

*Previous: [Chapter 3 — Advanced Lua](./chapter3_EN.md)*

---

## Table of Contents

1. [Game Data Acquisition](#1-game-data-acquisition)
2. [Custom Encounter Setup](#2-custom-encounter-setup)
3. [Custom Skills](#3-custom-skills)
4. [Lua — 광역 난사 (Area Barrage)](#4-lua--광역-난사-area-barrage)

---

## 1. Game Data Acquisition

Before writing any mod files, several values must be pulled from `dumpedData`. These values cannot be invented — they must match strings and IDs already present in the game.

### What was found and where

**Appearance string**

Located in `dumpedData/` under the identities section for character 1322:

```
1322_Pinky_Father2pAppearance
```

This string is a key into the game's character asset system. It maps to a specific 3D model, skeleton rig, and animation clip set. All `changemotion()` calls in Lua reference motion names (`"S1"`, `"S2"`, `"S3"`, etc.) that exist within this rig — if the appearance string is wrong, the rig will not have those motion names and animation calls will fail silently.

**Unit script ID**

Found by looking at an existing boss that shares the same movement and AI behavior:

```
8380
```

This integer is a key into the game's internal AI script table. The script defines the boss's pathfinding behavior, between-turn repositioning, and idle animation state. It is not a filename — it is a lookup ID. Copying it from a boss that uses the same character rig ensures compatible movement behavior.

**Map name and BGM**

Found inside `dumpedData/` encounter files:

```
mapName: "Cp9_HouseSpidersRoooftop"
bgmList: ["Battle_Cp9_Boss_5", "Battle_Cp9_Boss_6"]
```

`mapName` is a scene asset name loaded by the engine at encounter start. `bgmList` accepts multiple entries — the engine transitions from the first track to the second at a defined encounter threshold. Both strings must exactly match asset names registered in the game.

**Spawn position IDs**

Copied from an existing encounter using the same map:

```
allyPositionID:  175
enemyPositionID: 164
```

Each map has a set of predefined spawn point definitions identified by integer IDs. Using an ID that does not exist on the selected map will cause units to spawn at the origin or not at all.

**Base-game passive IDs**

Character 1322's passive set found in `dumpedData/`:

```
132219, 132224, 132220, 132222, 132223, 132214, 132230, 9999998
```

These passive IDs reference records that already exist in the game's passive database. They require no custom JSON files — the engine loads them from its own data. They are copied into `passiveSet.passiveIdList` in `soji.json` alongside the single custom passive `6820901`.

### ID scheme

All custom IDs were assigned to avoid collision with base-game data:

```
encounter:   682001
unit:        6820001
part:        6820101
passive:     6820901
skills:      682001010 … 682001200   (increments of 10)
```

---

## 2. Custom Encounter Setup

### Folder structure

```
BepInEx/plugins/Lethe/mods/
└── 소지아비로/
    ├── custom_encounters/
    │   └── 소지아비로/
    │       ├── encounter.json
    │       └── subchapterui.json
    ├── custom_limbus_data/
    │   ├── abnormality-unit/
    │   │   └── soji.json
    │   ├── abnormality-part/
    │   │   └── soji_part.json
    │   ├── passive/
    │   │   └── SojiPassive.json
    │   └── skill/
    │       └── soji_skills.json
    ├── custom_limbus_locale/
    │   └── KR/
    │       ├── enemyList/
    │       │   └── soji_locale.json
    │       └── skillList/
    │           └── soji_skills_locale.json
    └── modular_lua/
        ├── SojiMotion.lua
        ├── SojiBarrage.lua
        └── SojiPattern.lua
```

### encounter.json

```json
{
    "id": 682001,
    "stageLevel": 77,
    "stageType": "Abnormality",
    "isBatonPassOn": true,
    "participantInfo": { "min": 1, "max": 12 },
    "battleCameraInfo": {
        "defaultPosX": 0.0,
        "defaultPosY": 0.0,
        "defaultPosZ": 40.0
    },
    "waveList": [
        {
            "battleMapInfo": {
                "mapName": "Cp9_HouseSpidersRoooftop",
                "mapSize": 33.33
            },
            "unitList": [
                { "unitID": 6820001, "unitCount": 1, "unitLevel": 77 }
            ],
            "bgmList": ["Battle_Cp9_Boss_5", "Battle_Cp9_Boss_6"],
            "allyPositionID": 175,
            "enemyPositionID": 164
        }
    ],
    "staminaCost": 0,
    "turnLimit": 99,
    "rewardList": [],
    "effectiveResistCondition": 1.2
}
```

| Field | Notes |
|---|---|
| `id` | Must match `subchapterId` and `nodeId` in `subchapterui.json` |
| `stageType` | `"Abnormality"` activates boss-fight logic and the abnormality unit loading path |
| `isBatonPassOn` | Enables baton pass between identities mid-fight |
| `mapName` | Scene asset name — must exist in the game's asset registry |
| `bgmList` | Two-track — engine transitions to the second track at the encounter's midpoint threshold |
| `unitID` | Must match `id` in `soji.json` |
| `staminaCost` | 0 = free to enter |
| `effectiveResistCondition` | The resist multiplier is applied when the attacker's power-to-defense ratio exceeds this value |

### subchapterui.json

```json
{
    "chapterId": 2,
    "subchapterId": 682001,
    "nodeId": 682001,
    "isUnlockByUnlockCode": false,
    "unlockCode": 101,
    "relatedData": { "chapterId": 2, "subchapterId": 0, "nodeId": 0 },
    "uiConfig": {
        "customChapterText": "소지아비로",
        "chapterTagIconType": "BATTLE",
        "region": "p",
        "mapAreaId": 101,
        "timeLine": "PLAYGROUND"
    },
    "type": "STAGE_NODE"
}
```

`subchapterId` and `nodeId` must both equal `encounter.json`'s `id` — the engine uses these three values together to resolve the encounter when the node is selected. `timeLine: "PLAYGROUND"` places the node in the Playground area, which is always accessible regardless of story progression.

### soji.json (abnormality-unit)

```json
{
  "list": [{
    "id": 6820001,
    "appearance": "1322_Pinky_Father2pAppearance",
    "classType": "ZAYIN",
    "attributeType": "INDIGO",
    "unitScriptID": "8380",
    "startActionSlotNum": 7,
    "maxActionSlotNum": 7,
    "specialDuelRank": 5,
    "hp": { "defaultStat": 532, "incrementByLevel": 18 },
    "unitKeywordList": ["SMALL", "FINGER", "LITTLE_FINGER_FATHER_ENEMY"],
    "associationList": ["LITTLE_FINGER", "SPIDER_HOUSE"],
    "patternID": "PickByPattern_Abnormality_UptoActionSlotCnt",
    "abnormalityPartList": [6820101],
    "initBuffList": [],
    "hasMp": true,
    "panicType": 1070,
    "mentalConditionInfo": {
      "add": [{ "level": 1, "conditionIDList": [
        { "conditionID": "OnWinDuelAsParryingCountMultiply10AndPlus20Percent" }
      ]}],
      "min": [{ "level": 1, "conditionIDList": [
        { "conditionID": "OnLoseDuelAsParryingCountMultiply3AndPlus10Percent" }
      ]}]
    },
    "attributeList": [
      { "skillId": 682001010, "number": 0 },
      { "skillId": 682001020, "number": 0 },
      { "skillId": 682001030, "number": 0 },
      { "skillId": 682001040, "number": 0 },
      { "skillId": 682001060, "number": 0 },
      { "skillId": 682001080, "number": 0 },
      { "skillId": 682001100, "number": 0 },
      { "skillId": 682001110, "number": 0 },
      { "skillId": 682001130, "number": 0 },
      { "skillId": 682001140, "number": 0 },
      { "skillId": 682001160, "number": 0 },
      { "skillId": 682001170, "number": 0 },
      { "skillId": 682001180, "number": 0 },
      { "skillId": 682001190, "number": 0 },
      { "skillId": 682001200, "number": 0 }
    ],
    "passiveSet": {
      "passiveIdList": [132219, 132224, 132220, 132222, 132223, 132214, 132230, 9999998, 6820901]
    },
    "patternList": [ /* 6 patterns */ ]
  }]
}
```

| Field | Notes |
|---|---|
| `appearance` | Asset key for the character model, rig, and animation clip set |
| `classType` | Boss threat tier — affects UI display and certain game system calculations |
| `attributeType` | The unit's primary sin attribute — used for damage type matching and resistance calculations |
| `unitScriptID` | Integer key into the AI script table — controls movement and idle behavior |
| `startActionSlotNum` | Number of skill slots active at encounter start |
| `maxActionSlotNum` | Maximum skill slots the boss can have (set equal to `startActionSlotNum` for a fixed slot count) |
| `specialDuelRank` | The boss's clash power tier. Higher values cause the boss to win clashes against lower-ranked units more frequently |
| `hp.incrementByLevel` | HP added per level above the base level |
| `patternID` | `PickByPattern_Abnormality_UptoActionSlotCnt` — selects patterns from `patternList` up to the count of active action slots |
| `hasMp` | Enables the stagger/MP system — the boss has a stagger bar that can be broken |
| `panicType` | Determines the panic animation and behavior state when the boss's SP reaches 0 |
| `abnormalityPartList` | List of part IDs attached to this unit. Must match `id` values in `soji_part.json` |
| `attributeList` | Every skill the boss may use across all patterns must be listed here. `"number": 0` is correct for boss units — bosses do not consume sin resources |
| `passiveSet` | IDs `132219`-`132230` and `9999998` are base-game passives from dumpedData. `6820901` is the custom Lua-trigger passive |

**`mentalConditionInfo`** controls SP (sanity) gain and loss on clash outcomes. The `add` list fires on SP gain events (winning a duel), `min` fires on SP loss events (losing a duel). The condition ID encodes the formula directly in its name:

- `OnWinDuelAsParryingCountMultiply10AndPlus20Percent` — SP gain = (parry count × 10 + 20)%
- `OnLoseDuelAsParryingCountMultiply3AndPlus10Percent` — SP loss = (parry count × 3 + 10)%

### soji_part.json (abnormality-part)

```json
{
  "list": [{
    "id": 6820101,
    "hp": { "defaultStat": 532, "incrementByLevel": 18 },
    "minSpeedList": [2],
    "maxSpeedList": [5],
    "partType": "BODY",
    "isDestroyable": "false",
    "spreadMpEffectToAbnormality": "true",
    "resistInfo": {
      "atkResistList": [
        { "type": "SLASH",     "value": 1.2 },
        { "type": "PENETRATE", "value": 1.0 },
        { "type": "HIT",       "value": 1.0 }
      ],
      "attributeResistList": [
        { "type": "CRIMSON",  "value": 1.2 },
        { "type": "SCARLET",  "value": 1.0 },
        { "type": "AMBER",    "value": 0.8 },
        { "type": "SHAMROCK", "value": 0.8 },
        { "type": "AZURE",    "value": 1.2 },
        { "type": "INDIGO",   "value": 0.8 },
        { "type": "VIOLET",   "value": 1.0 },
        { "type": "WHITE",    "value": 2.0 },
        { "type": "BLACK",    "value": 2.0 }
      ]
    },
    "passiveSet": { "passiveIdList": [132205] }
  }]
}
```

The part `id` (`6820101`) must match the value in `abnormalityPartList` in `soji.json`. `isDestroyable: "false"` means the part's HP bar can reach 0 but does not trigger a destruction event — the encounter continues. `spreadMpEffectToAbnormality: "true"` propagates MP (stagger) damage applied to the part up to the main abnormality unit, so hitting the part also fills the main unit's stagger gauge.

Resistance values: `< 1.0` weak (damage multiplied up), `1.0` normal, `> 1.0` resistant (damage multiplied down). `WHITE` and `BLACK` at `2.0` means those attribute types deal half effective damage.

### SojiPassive.json

```json
{
  "list": [
    {
      "id": 6820901,
      "attributeStockCondition": [],
      "requireIDList": [
        "Modular/TIMING:AfterSlots/LUA:SojiPattern/LUAMAIN:onSkillUse"
      ]
    }
  ]
}
```

This passive has no stat effects. Its only function is to register a Lua trigger through `requireIDList`. The string is parsed by Lethe's ModularSkillScripts system:

```
Modular / TIMING:AfterSlots / LUA:SojiPattern / LUAMAIN:onSkillUse
  │              │                  │                    │
  │              │                  │                    └─ Function name to call in the file
  │              │                  └─ Lua file: modular_lua/SojiPattern.lua
  │              └─ Timing: fires after skill slots are assigned, before skills execute
  └─ ModularSkillScripts system prefix
```

`AfterSlots` fires once per round after the pattern selector assigns skills to slots, before any skill begins executing. This is the only timing window in which `skillslotreplace()` is effective — after this point, slot assignments are locked.

The passive is active because its ID (`6820901`) is listed in `passiveSet.passiveIdList` in `soji.json`.

### Locale

**soji_locale.json** (`custom_limbus_locale/KR/enemyList/`):

```json
{
  "6820001": {
    "a": 0,
    "id": "6820001",
    "name": "소지 아비",
    "desc": ""
  }
}
```

The top-level key is the unit ID as a string. This format is specific to `enemyList` locale files — do not use the `dataList` array format used for other locale types.

**soji_skills_locale.json** (`custom_limbus_locale/KR/skillList/`):

```json
{
  "682001020": {
    "id": "682001020",
    "levelList": [{
      "level": 1,
      "name": "이연잔",
      "desc": "[적중시] 호흡 1 획득.",
      "coinlist": [
        { "coindescs": [{ "desc": "[적중시] 호흡 +1", "summary": "" }] },
        { "coindescs": [{ "desc": "[적중시] 호흡 +1", "summary": "" }] },
        { "coindescs": [{ "desc": "[적중시] 호흡 +1", "summary": "" }] },
        { "coindescs": [{ "desc": "[적중시] 호흡 +1", "summary": "" }] },
        { "coindescs": [{ "desc": "[적중시] 호흡 +1", "summary": "" }] }
      ]
    }]
  }
}
```

`coinlist` must have exactly as many entries as the skill has coins — the engine maps each entry by index to its corresponding coin. A mismatch in count causes coin descriptions to display incorrectly or not at all.

---

## 3. Custom Skills

### 3-1. 연격 (682001010)

```json
{
  "id": 682001010,
  "skillTier": 1,
  "skillType": "SKILL",
  "skillData": [{
    "attributeType": "VIOLET",
    "atkType": "SLASH",
    "defType": "ATTACK",
    "targetNum": 5,
    "skillTargetType": "RANDOM",
    "canDuel": true,
    "canChangeTarget": true,
    "skillLevelCorrection": 0,
    "defaultValue": 10,
    "skillMotion": "S1",
    "viewType": "ENCOUNTER",
    "parryingCloseType": "NEAR",
    "abilityScriptList": [
      { "scriptName": "EmptyBody" },
      {
        "scriptName": "GiveBuffOnUse",
        "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 0, "turn": 2 }
      }
    ],
    "coinList": [
      {
        "operatorType": "ADD", "scale": 4,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame0" }
        ]
      },
      {
        "operatorType": "ADD", "scale": 4,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame0" }
        ]
      },
      {
        "operatorType": "ADD", "scale": 4,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame0" }
        ]
      },
      {
        "operatorType": "ADD", "scale": 7,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 2, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame1" }
        ]
      }
    ]
  }]
}
```

**Core fields:**

| Field | Value | Effect |
|---|---|---|
| `defType` | `"ATTACK"` | Damage is calculated using the unit's attack stat. `"DEFENSE"` would use the defense stat instead |
| `targetNum` | 5 | Number of hit instances distributed across targets |
| `skillTargetType` | `"RANDOM"` | Hits are distributed randomly among available targets |
| `skillLevelCorrection` | 0 | Additional base power added per unit level. `0` means power scales only from level-based stat growth |
| `defaultValue` | 10 | Base coin power before `scale` is added |
| `parryingCloseType` | `"NEAR"` | The skill can only target units in the near range — affects clash eligibility |
| `canDuel` | true | This skill can enter clash resolution against opposing skills |
| `viewType` | `"ENCOUNTER"` | Camera pulls back to show all units. Use `"BATTLE"` for single-target clashes |

**Top-level `abilityScriptList`** — fires once when the skill is selected, before any coin resolves:

- `EmptyBody` — required declaration on every boss skill. Without it the attack body is undefined and the skill will not execute correctly.
- `GiveBuffOnUse` with `stack: 0, turn: 2` — `stack: 0` does not add stacks. It only refreshes the Breath buff's remaining duration to 2 turns. The current stack count is preserved.

**Coins:**

| Coin | `scale` | Motion | Breath on hit |
|---|---|---|---|
| [0] [1] [2] | 4 | S1 frame 0 | +1 |
| [3] | 7 | S1 frame 1 | +2 |

`operatorType: "ADD"` means each coin adds its `scale` to the running power total. The total at the time the coin resolves is the final attack power for that hit. `GiveBuffOnSucceedAttack` with `turn: 0` means the Breath buff has no turn-based expiry — it persists until removed by another effect.

---

### 3-2. 이연잔 (682001020)

```json
{
  "id": 682001020,
  "skillTier": 1,
  "skillType": "SKILL",
  "skillData": [{
    "attributeType": "VIOLET",
    "atkType": "SLASH",
    "defType": "ATTACK",
    "targetNum": 1,
    "skillTargetType": "RANDOM",
    "canDuel": true,
    "canChangeTarget": true,
    "skillLevelCorrection": 0,
    "defaultValue": 10,
    "skillMotion": "S2",
    "viewType": "BATTLE",
    "parryingCloseType": "NEAR",
    "abilityScriptList": [
      { "scriptName": "EmptyBody" },
      {
        "scriptName": "GiveBuffOnUse",
        "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 0, "turn": 3 }
      }
    ],
    "coinList": [
      {
        "operatorType": "ADD", "scale": 3,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s2_frame1" }
        ]
      },
      {
        "operatorType": "ADD", "scale": 3,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s2_frame1" }
        ]
      },
      {
        "operatorType": "ADD", "scale": 3,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s2_frame2" }
        ]
      },
      {
        "operatorType": "ADD", "scale": 3,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s2_frame1" }
        ]
      },
      {
        "operatorType": "ADD", "scale": 3,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s2_frame2" }
        ]
      }
    ]
  }]
}
```

5 coins, all `scale: 3`. Each coin grants Breath +1 on hit. Unlike 682001010, this skill uses `viewType: "BATTLE"` because `targetNum: 1` makes it a single-target clash skill — the camera stays in close-up.

`GiveBuffOnUse` at top level with `stack: 0, turn: 3` refreshes Breath duration to 3 turns on selection without adding stacks — the same duration-refresh pattern used in 682001010.

The S2 animation has 3 frames. Coins call `s2_frame1` or `s2_frame2` from `SojiMotion.lua` at `TIMING:ChangeMotion`:

| Coin | Motion |
|---|---|
| [0] [1] [3] | S2 frame 1 |
| [2] [4] | S2 frame 2 |

---

### 3-3. 광역 난사 (682001200)

```json
{
  "id": 682001200,
  "skillTier": 3,
  "skillType": "SKILL",
  "skillData": [{
    "attributeType": "INDIGO",
    "atkType": "SLASH",
    "defType": "ATTACK",
    "targetNum": 5,
    "skillTargetType": "RANDOM",
    "canDuel": true,
    "canChangeTarget": true,
    "skillLevelCorrection": 4,
    "defaultValue": 9,
    "importanceLevel": 3,
    "skillMotion": "S1",
    "viewType": "ENCOUNTER",
    "parryingCloseType": "NEAR",
    "abilityScriptList": [
      { "scriptName": "EmptyBody" },
      { "scriptName": "DealRandomTargetAmongTargetsMTFirst" },
      { "scriptName": "CanBeChangedTargetIgnoreSpeedIfEnemy" },
      { "scriptName": "Modular/TIMING:BeforeUse/makeunbreakable(1)" }
    ],
    "coinList": [
      /* coins [0]-[6]: barrageRandomS2 */
      /* coin  [7]:     s6_frame0       */
      /* coin  [8]:     s3_frame0, Unbreakable, Breath reward */
    ]
  }]
}
```

**Top-level scripts:**

| Script | Effect |
|---|---|
| `EmptyBody` | Required boss attack body declaration |
| `DealRandomTargetAmongTargetsMTFirst` | Distributes 9 hit instances across available targets. Prioritizes targets that are already being targeted by the most allies (MT = most-targeted first), spreading pressure efficiently across the team |
| `CanBeChangedTargetIgnoreSpeedIfEnemy` | Normally, target switching during skill resolution is gated by the unit's speed stat. This script removes that restriction, allowing the boss to redirect hits to any target freely regardless of speed order |
| `makeunbreakable(1)` at `BeforeUse` | The integer parameter specifies how many coins from the end of the skill become Unbreakable. `makeunbreakable(1)` makes the final coin (coin 8) unbreakable |

`importanceLevel: 3` marks this as a high-priority skill for the AI's slot assignment logic. `skillLevelCorrection: 4` adds 4 base power per unit level, making the skill significantly stronger at higher levels than skills with `skillLevelCorrection: 0`.

**Coins [0]-[6] — random S2 frame via Lua:**

```json
{
  "operatorType": "ADD",
  "scale": 4,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiBarrage/LUAMAIN:barrageRandomS2" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5 } }
  ]
}
```

**Coin [7] — fixed S6 frame:**

```json
{
  "operatorType": "ADD",
  "scale": 4,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s6_frame0" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5 } }
  ]
}
```

**Coin [8] — Unbreakable final blow:**

```json
{
  "operatorType": "ADD",
  "scale": 7,
  "grade": 2,
  "color": "GREY",
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s3_frame0" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5 } },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 3, "turn": 0 } }
  ]
}
```

`grade: 2, color: "GREY"` is the JSON-level declaration for the Unbreakable visual marker — it makes the coin render as a grey Unbreakable coin in the UI. The actual unbreakable behavior is enforced by `makeunbreakable(1)` at `BeforeUse`. Coin 8 has `scale: 7` (vs `scale: 4` on the others), making its power contribution higher. It also grants Breath +3 to self on hit — the boss recovers Breath after completing the full barrage.

---

## 4. Lua — 광역 난사 (Area Barrage)

<video src="https://github.com/AIGhostWriter/Lethe_Guide/raw/main/KakaoTalk_20260516_162012462.mp4" controls width="100%"></video>

### 4-1. Why Lua Is Needed

`광역 난사` (682001200) has 9 coins. The S2 animation has 3 frames. Coins 0-6 each play a randomly selected frame per hit. JSON has no `random()` function — Lua is required.

### 4-2. SojiBarrage.lua

```lua
function barrageRandomS2()
    local frame = random(0, 2)
    changemotion("S2", frame)
    log("[SojiBarrage] S2[" .. frame .. "]", 0)
end
```

- `random(0, 2)` — Lethe's built-in RNG function. Returns an integer in the range [0, 2] inclusive with uniform distribution. Called fresh each coin, so each of the 7 coins gets an independent random frame.
- `changemotion(motionName, frameIndex)` — sets the animation clip frame for the currently resolving coin. `"S2"` is the motion name within the appearance rig. This function is only valid at `TIMING:ChangeMotion` — calls outside this timing are ignored.

Each of coins 0-6 registers this function as a `ChangeMotion` script in its `abilityScriptList`:

```json
{ "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiBarrage/LUAMAIN:barrageRandomS2" }
```

The Lethe ModularSkillScripts system parses this string: `LUA:SojiBarrage` resolves to `modular_lua/SojiBarrage.lua`, and `LUAMAIN:barrageRandomS2` calls the function `barrageRandomS2` in that file. Coins 7 and 8 use fixed functions from `SojiMotion.lua` (`s6_frame0`, `s3_frame0`) instead of the random function.


### 4-3. The Vibration Chain

Every coin executes the same four-step sequence. The order within `abilityScriptList` is the execution order:

```
① SuperCoin                              → flip bypassed, attack resolves unconditionally
② ChangeMotion (Lua)                     → animation frame assigned before the hit renders
③ VibrationExplosionOnSucceedAttack      → detonates current Vibration stacks on target (fires on hit)
④ GiveBuffOnSucceedAttack: Vibration 5/5 → applies 5 stacks of Vibration with 5-turn duration (fires on hit)
```

Step ③ fires before step ④ because `abilityScriptList` entries execute in array order. The explosion consumes the stacks that existed before this coin, then step ④ reloads 5 stacks for the next coin to consume.

Result across all 9 coins:

```
Coin 0: explode (0 stacks)  → apply Vibration 5
Coin 1: explode (5 stacks)  → apply Vibration 5
Coin 2: explode (5 stacks)  → apply Vibration 5
...
Coin 7: explode (5 stacks), S6[0]  → apply Vibration 5
Coin 8: explode (5 stacks), S3[0]  → apply Vibration 5  +  Breath 3 to self
```

Coin 0 explodes at 0 stacks because no Vibration was applied before the skill started. From coin 1 onward, every explosion fires at the full 5 stacks loaded by the previous coin.

---

*For the full Lua function reference and TIMING keyword list, see [Chapter 3](./chapter3_EN.md).*
