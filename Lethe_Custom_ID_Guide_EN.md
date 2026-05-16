# Lethe Custom Identity Guide
### Using SojiAbiIdentity as a running example

---

## Table of Contents

| Chapter | Topic |
|---------|-------|
| **1** | Identity Foundation — `personality.json`, folder layout, ID scheme, appearance |
| **2** | Player Skills — flip coins, SuperCoin, target types, attribute rules |
| **3** | Player Passives — `pp.json`, `passive.json`, base-game passive IDs, Lua triggers |

---

# Chapter 1 · Identity Foundation

## 1.1 Folder Structure

An identity mod uses a **completely different folder tree** from an encounter mod.
Encounter mods live under `custom_limbus_data/abnormality-unit/`.
Identity mods live under `custom_limbus_data/personality/`.

```
SojiAbiIdentity/
├── custom_limbus_data/
│   ├── personality/
│   │   └── sojiabi_personality.json        ← unit definition
│   ├── skill/
│   │   └── sojiabi_id_skills.json          ← player skills
│   ├── passive/
│   │   └── sojiabi_id_passive.json         ← passive effect definitions
│   └── personality-passive/
│       └── sojiabi_pp.json                 ← wires passives to the identity
├── custom_limbus_locale/
│   └── KR/
│       ├── personalityList/
│       │   └── sojiabi_personality.json    ← identity name / flavor
│       ├── skillList/
│       │   └── sojiabi_id_skills_locale.json
│       └── passiveList/
│           └── sojiabi_id_passive_locale.json
└── modular_lua/
    ├── SojiIDPassive.lua
    ├── SojiMotion.lua
    └── SojiBarrage.lua
```

> ⚠️ **Encounter vs Identity — missing the `personality-passive/` folder is the single most common mistake.**
> Encounter units declare passives inside the unit JSON (`"passiveSet"`).
> Identities use a *separate* `personality-passive/*.json` file to wire passives.
> Without it, every passive you define is silently ignored.

---

## 1.2 ID Scheme

Choose a 7-digit base ID that does not collide with base-game IDs (which are 6 digits or fewer).

```
6821000   ← personality (identity) ID
6821010   ← skill tier 1
6821020   ← skill tier 2
6821030   ← skill tier 3
6821100   ← defense skill
6821901   ← battle passive
6821921   ← supporter passive
```

The personality ID (`6821000`) must match across:
- `personality.json`  →  `"id": 6821000`
- `sojiabi_pp.json`   →  `"personalityID": 6821000`
- Locale file key     →  `"6821000": { … }`

---

## 1.3 `personality.json` — Full Field Reference

```json
{
  "list": [
    {
      "id": 6821000,
      "appearance": "1322_Pinky_Father2pAppearance",
      "unitKeywordList": ["SMALL", "FINGER"],
      "associationList": ["LITTLE_FINGER", "SPIDER_HOUSE"],
      "skillKeywordList": ["Breath", "PhantomIncision"],
      "characterId": 10,
      "panicType": 1070,
      "defenseSkillIDList": [6821100],
      "slotWeightConditionList": ["Default1"],
      "rank": 3,
      "hp": {
        "defaultStat": 400,
        "incrementByLevel": 15
      },
      "defCorrection": 3,
      "minSpeedList": [3, 3, 3, 3],
      "maxSpeedList": [7, 7, 7, 7],
      "uniqueAttribute": "VIOLET",
      "mentalConditionInfo": {
        "add": [
          { "level": 1, "conditionIDList": [{ "conditionID": "OnWinDuelAsParryingCountMultiply5AndPlus10Percent" }] }
        ],
        "min": [
          { "level": 1, "conditionIDList": [{ "conditionID": "OnLoseDuelAsParryingCountMultiply3AndPlus10Percent" }] }
        ]
      },
      "breakSection": { "sectionList": [60] },
      "resistInfo": {
        "atkResistList": [
          { "type": "SLASH",     "value": 1.2 },
          { "type": "PENETRATE", "value": 0.8 },
          { "type": "HIT",       "value": 1.0 }
        ]
      },
      "attributeList": [
        { "skillId": 6821010, "number": 3 },
        { "skillId": 6821020, "number": 2 },
        { "skillId": 6821030, "number": 1 },
        { "skillId": 6821100, "number": 1 }
      ]
    }
  ]
}
```

### Field-by-field breakdown

| Field | Type | Notes |
|-------|------|-------|
| `id` | int | Must match `personalityID` in `pp.json` |
| `appearance` | string | Asset string from base-game dumpedData. Must be exact — wrong string → wrong model. See §1.4 |
| `unitKeywordList` | string[] | Unit tags used by game systems (e.g. team buffs). Can be empty `[]` |
| `associationList` | string[] | Association tags. Used by support passives and ally effects |
| `skillKeywordList` | string[] | Declares which buff keywords this unit "owns" — drives tooltip badge display. List every custom buff keyword the unit gives to itself |
| `characterId` | int | Which sinner's body to bind to. 1=Yi Sang … 10=Sinclair … 12=Gregor |
| `panicType` | int | Panic EGO ID. Must be a valid base-game ID or the game crashes on panic |
| `defenseSkillIDList` | int[] | IDs of counter/evade skills. Must reference IDs defined in your skill JSON |
| `slotWeightConditionList` | string[] | Slot weight preset. `"Default1"` is safe for all 3-skill identities |
| `rank` | int | Star rank: 1, 2, or 3 |
| `hp.defaultStat` | int | Base HP at level 1 |
| `hp.incrementByLevel` | int | HP added per level |
| `defCorrection` | int | Defense level modifier. Positive = tankier |
| `minSpeedList` | int[4] | Minimum speed for each of the 4 skill slots |
| `maxSpeedList` | int[4] | Maximum speed for each of the 4 skill slots |
| `uniqueAttribute` | string | The identity's sin attribute. Valid: `CRIMSON SCARLET AMBER SHAMROCK INDIGO VIOLET` |
| `breakSection.sectionList` | int[] | HP % thresholds that trigger the break/stagger animation |
| `resistInfo.atkResistList` | object[] | Attack type resistances. `value: 1.0` = neutral, `>1.0` = resistant, `<1.0` = weak |
| `attributeList` | object[] | Links skill IDs to slot positions. `number` = how many sin attribute resonance dice the skill contributes |

### `attributeList` — slot assignment

```json
"attributeList": [
  { "skillId": 6821010, "number": 3 },   ← skill 1, contributes 3 sin dice
  { "skillId": 6821020, "number": 2 },   ← skill 2, contributes 2 sin dice
  { "skillId": 6821030, "number": 1 },   ← skill 3, contributes 1 sin die
  { "skillId": 6821100, "number": 1 }    ← defense skill, contributes 1 sin die
]
```

The order of entries determines which skill slot each skill occupies.

---

## 1.4 The `appearance` String

This is the single most error-prone field. The string must match a model asset bundled in the game.

**How to find a valid appearance string:**

Open `dumpedData/limbus_data/personality/*.json` for any base-game identity and copy its `appearance` value.

For a Soji Abi-themed identity:
```
"1322_Pinky_Father2pAppearance"
```
This string is used by both base-game identities `104923` and `111923` and works reliably.

> ⚠️ **Wrong `appearance` = wrong model rendered in game.**
> The game will not crash — it silently loads a fallback model.
> Always verify in-game immediately after first load.

| ❌ Wrong | ✅ Correct |
|---------|-----------|
| `"1145_Thumb_LeiHengAppearance"` | `"1322_Pinky_Father2pAppearance"` |
| String from enemy/abnormality unit | String from a `dumpedData/personality` JSON |

Encounter mods use `appearance` in `abnormality-unit`. The exact same string format works for identities — the asset pipeline is shared.

---

## 1.5 Locale — `personalityList`

```json
{
  "6821000": {
    "name": "소지 아비",
    "desc": "거미 집의 새끼손가락 아비",
    "summary": "",
    "flavor": ""
  }
}
```

Key is the personality ID as a string. The file lives at:
```
custom_limbus_locale/KR/personalityList/sojiabi_personality.json
```

---

# Chapter 2 · Player Skills

## 2.1 Core Structural Differences from Encounter Skills

Encounter skills and identity skills share the same JSON schema, but player skills have strict rules that encounter skills do not.

| Property | Encounter | Identity (Player) |
|----------|-----------|-------------------|
| `coinList` coin color | Any / omit | **Must be `"GOLD"` for flip coins** |
| `coinList` coin grade | Any / omit | **Must be `1` for flip coins** |
| `prob` | Omit (always heads) | **Must be `0.5` for flip coins** |
| `skillTargetType` | `RANDOM`, `FRONT`, etc. | Recommend `FRONT`; `RANDOM` works differently for players |
| `attributeType` | Any including `CRIMSON` | **`CRIMSON` is invalid → crash** |
| `defType` on attack skills | `ATTACK` | `ATTACK` |
| `defType` on defense skills | `COUNTER` or `EVADE` | `COUNTER` or `EVADE` |
| `viewType` | Flexible | Use `BATTLE` for normal, `ENCOUNTER` for the big skill animation |

> ⚠️ **CRIMSON crash** — `attributeType: "CRIMSON"` in a player skill will crash the game on combat start.
> Valid player sin attributes: `CRIMSON` is boss-only. Use `VIOLET`, `INDIGO`, `AMBER`, `SCARLET`, `SHAMROCK` only.

---

## 2.2 Skill JSON Structure

```json
{
  "list": [
    {
      "id": 6821010,
      "skillTier": 1,
      "skillType": "SKILL",
      "skillData": [
        {
          "gaksungLevel": 1,
          "attributeType": "VIOLET",
          "atkType": "SLASH",
          "defType": "ATTACK",
          "targetNum": 1,
          "skillTargetType": "FRONT",
          "canTeamKill": false,
          "canDuel": true,
          "canChangeTarget": true,
          "skillLevelCorrection": -1,
          "defaultValue": 8,
          "skillMotion": "S2",
          "viewType": "BATTLE",
          "parryingCloseType": "NEAR",
          "abilityScriptList": [ … ],
          "coinList": [ … ]
        }
      ]
    }
  ]
}
```

### Top-level skill fields

| Field | Notes |
|-------|-------|
| `id` | Must match an entry in `personality.json` → `attributeList` |
| `skillTier` | 1 = skill 1, 2 = skill 2, 3 = skill 3. Defense skills also use 1 |
| `skillType` | Always `"SKILL"` for player identities |

### `skillData` fields

| Field | Type | Notes |
|-------|------|-------|
| `gaksungLevel` | int | Awakening level the data applies to. Use `1` for the base version |
| `attributeType` | string | Sin attribute. **Not CRIMSON** |
| `atkType` | string | `SLASH`, `PENETRATE`, `HIT` |
| `defType` | string | `ATTACK` for offensive skills, `COUNTER`/`EVADE` for defense |
| `targetNum` | int | How many targets the skill hits (1 for single, 5 for multi-hit) |
| `skillTargetType` | string | `FRONT` targets frontmost enemy. `RANDOM` hits random — for multi-hit player skills this behaves differently than encounter |
| `canTeamKill` | bool | `false` always for player |
| `canDuel` | bool | `true` to enable clashing |
| `canChangeTarget` | bool | `true` allows retargeting when original target dies |
| `skillLevelCorrection` | int | Flat bonus/penalty to clash power from skill level. Negative makes it weaker |
| `defaultValue` | int | Base power at level 1 |
| `skillMotion` | string | Animation set to use. `"S1"` through `"S6"` correspond to the sinner's motion library |
| `viewType` | string | `"BATTLE"` = normal size. `"ENCOUNTER"` = cinematic scale (use sparingly, on 2-skill only) |
| `parryingCloseType` | string | `"NEAR"` for melee |

---

## 2.3 Flip Coins

Every coin a player can flip must have these three fields:

```json
{
  "operatorType": "ADD",
  "color": "GOLD",
  "grade": 1,
  "prob": 0.5,
  "scale": 6,
  "abilityScriptList": [ … ]
}
```

| Field | Value | Meaning |
|-------|-------|---------|
| `color` | `"GOLD"` | Flip coin (shows heads/tails) |
| `grade` | `1` | Standard flip coin weight |
| `prob` | `0.5` | 50% chance of heads |
| `scale` | int | Power added when this coin lands heads |

> ⚠️ **Omitting `color`, `grade`, or `prob` produces a non-flipping coin.**
> The coin will always count as heads and never animate.
> This was the cause of all broken encounter-ported skills — encounter skills omit these fields because encounters never flip.

**Example — Skill 1 (이연잔), 5 flip coins:**

```json
"coinList": [
  {
    "operatorType": "ADD", "color": "GOLD", "grade": 1, "prob": 0.5, "scale": 6,
    "abilityScriptList": [
      { "scriptName": "GiveBuffOnSucceedAttack",
        "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 1, "activeRound": 0 } },
      { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s2_frame1" }
    ]
  },
  … (repeat for each coin)
]
```

`GiveBuffOnSucceedAttack` fires when the coin lands **heads AND the attack hits**. `turn: 1` keeps the buff alive for one round — `turn: 0` can cause the buff to expire immediately for some buff types.

---

## 2.4 SuperCoin in Player Skills

SuperCoins are coins that are always heads and deal high damage. They are used in the 9-coin barrage skill (광역난사) to simulate a full-auto burst.

```json
{
  "operatorType": "ADD",
  "scale": 8,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiBarrage/LUAMAIN:barrageRandomS2" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5, "activeRound": 0 } }
  ]
}
```

SuperCoin coins **do not** need `color`, `grade`, or `prob`.
The final coin in the barrage is the finisher and uses `color: "GREY", grade: 2` for the visual indicator:

```json
{
  "operatorType": "ADD", "scale": 12,
  "grade": 2,
  "color": "GREY",
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s3_frame0" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5, "activeRound": 0 } },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 3, "turn": 1, "activeRound": 0 } }
  ]
}
```

**Full 9-coin barrage structure for 광역난사 (Skill 2):**

```json
{
  "id": 6821020,
  "skillTier": 2,
  "skillType": "SKILL",
  "skillData": [
    {
      "gaksungLevel": 1,
      "attributeType": "INDIGO",
      "atkType": "SLASH",
      "defType": "ATTACK",
      "targetNum": 5,
      "skillTargetType": "FRONT",
      "canTeamKill": false,
      "canDuel": true,
      "canChangeTarget": true,
      "skillLevelCorrection": 4,
      "defaultValue": 15,
      "importanceLevel": 3,
      "skillMotion": "S1",
      "viewType": "ENCOUNTER",
      "parryingCloseType": "NEAR",
      "abilityScriptList": [
        { "scriptName": "EmptyBody" },
        { "scriptName": "Modular/TIMING:BeforeUse/makeunbreakable(1)" }
      ],
      "coinList": [
        { "operatorType": "ADD", "scale": 8, "abilityScriptList": [
            { "scriptName": "SuperCoin" },
            { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiBarrage/LUAMAIN:barrageRandomS2" },
            { "scriptName": "VibrationExplosionOnSucceedAttack" },
            { "scriptName": "GiveBuffOnSucceedAttack",
              "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5, "activeRound": 0 } }
        ]},
        … (coins 2–8 identical to coin 1),
        { "operatorType": "ADD", "scale": 12, "grade": 2, "color": "GREY",
          "abilityScriptList": [
            { "scriptName": "SuperCoin" },
            { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s3_frame0" },
            { "scriptName": "VibrationExplosionOnSucceedAttack" },
            { "scriptName": "GiveBuffOnSucceedAttack",
              "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5, "activeRound": 0 } },
            { "scriptName": "GiveBuffOnSucceedAttack",
              "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 3, "turn": 1, "activeRound": 0 } }
        ]}
      ]
    }
  ]
}
```

> ⚠️ **`makeunbreakable(1)` in `abilityScriptList` (skill level, not coin level)** prevents the skill from being interrupted mid-barrage. Without it, a clash loss during the barrage cancels remaining coins.

---

## 2.5 Defense Skill

Defense skills use `defType: "COUNTER"` (or `"EVADE"`). They must be listed in `defenseSkillIDList` in `personality.json`.

```json
{
  "id": 6821100,
  "skillTier": 1,
  "skillType": "SKILL",
  "skillData": [
    {
      "attributeType": "AMBER",
      "atkType": "SLASH",
      "defType": "COUNTER",
      "targetNum": 1,
      "skillTargetType": "FRONT",
      "canTeamKill": false,
      "canDuel": true,
      "canChangeTarget": true,
      "skillLevelCorrection": 0,
      "defaultValue": 9,
      "skillMotion": "S6",
      "viewType": "BATTLE",
      "parryingCloseType": "NEAR",
      "abilityScriptList": [
        { "scriptName": "EmptyBody" }
      ],
      "coinList": [
        {
          "operatorType": "ADD", "color": "GOLD", "grade": 1, "prob": 0.5, "scale": 8,
          "abilityScriptList": [
            { "scriptName": "GiveBuffOnSucceedAttack",
              "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 1, "activeRound": 0 } }
          ]
        },
        {
          "operatorType": "ADD", "color": "GOLD", "grade": 1, "prob": 0.5, "scale": 8,
          "abilityScriptList": [
            { "scriptName": "GiveBuffOnSucceedAttack",
              "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 2, "turn": 1, "activeRound": 0 } }
          ]
        }
      ]
    }
  ]
}
```

Defense skills do not need `gaksungLevel` — the game resolves them without an awakening level check.

---

## 2.6 Skill-Level `abilityScriptList` vs Coin-Level

There are **two** `abilityScriptList` arrays in every skill:

```
skillData
  └── abilityScriptList     ← fires ONCE when skill is used (regardless of coins)
  └── coinList
        └── abilityScriptList   ← fires PER COIN that lands heads + hits
```

Common skill-level scripts:

| Script | Effect |
|--------|--------|
| `EmptyBody` | Required placeholder. Always include as first entry |
| `GiveBuffOnUse` | Gives a buff once when skill activates |
| `ReuseOnKillOrBreakTargetToRandomTarget` | Re-triggers the skill on kill (필연쇄 mechanic) |
| `Modular/TIMING:BeforeUse/makeunbreakable(1)` | Prevents clash interruption (barrage skills) |

Common coin-level scripts:

| Script | Effect |
|--------|--------|
| `GiveBuffOnSucceedAttack` | Gives buff on heads + hit |
| `VibrationExplosionOnSucceedAttack` | Detonates Vibration stacks on hit |
| `SuperCoin` | Forces this coin to always land heads |
| `Modular/TIMING:ChangeMotion/LUA:X/LUAMAIN:Y` | Plays a Lua-driven animation frame |

---

## 2.7 `GiveBuffOnUse` — Self Buff on Skill Use

```json
{
  "scriptName": "GiveBuffOnUse",
  "buffData": {
    "buffKeyword": "Breath",
    "target": "Self",
    "stack": 2,
    "turn": 3,
    "activeRound": 0
  }
}
```

> ⚠️ **`stack: 0` does nothing if the buff doesn't already exist.**
> `stack: 0, turn: 3` only refreshes the duration on an existing buff — if the character has 0 Breath stacks, this is a no-op.
> Always use `stack: 2` or higher to actually grant the buff.

---

## 2.8 Skill Locale

```json
{
  "6821010": {
    "name": "이연잔",
    "desc": "코인 적중 시 호흡 횟수 +1",
    "summary": ""
  },
  "6821020": {
    "name": "광역난사",
    "desc": "9코인 연속 공격. 적중 시 진동 5 부여 및 진동 폭발.",
    "summary": ""
  },
  "6821030": {
    "name": "필연쇄",
    "desc": "적중 시 진동 + 잔상베기 부여. 처치 시 재발동.",
    "summary": ""
  },
  "6821100": {
    "name": "방어",
    "desc": "코인 성공 시 호흡 횟수 +1~2",
    "summary": ""
  }
}
```

Key = skill ID as string. Any skill ID present in `personality.json → attributeList` needs a locale entry or the game shows a blank name.

---

# Chapter 3 · Player Passives

## 3.1 The Two-File System

Encounter passives are declared in one place: the unit JSON's `"passiveSet"`.
Identity passives require **two** files:

```
passive/sojiabi_id_passive.json         ← defines what each passive ID does
personality-passive/sojiabi_pp.json     ← wires passive IDs to the identity
```

Missing either file means zero passives in game — no error, no warning, just silence.

---

## 3.2 `sojiabi_pp.json` — Wiring Passives to the Identity

```json
{
  "list": [
    {
      "personalityID": 6821000,
      "battlePassiveList": [
        {
          "level": 1,
          "passiveIDList": [132219, 132220, 132222, 132223, 132214, 132230, 6821901]
        }
      ],
      "supporterPassiveList": [
        {
          "level": 3,
          "passiveIDList": [6821921]
        }
      ]
    }
  ]
}
```

| Field | Notes |
|-------|-------|
| `personalityID` | Must match `personality.json → id` |
| `battlePassiveList[].level` | Awakening level at which these passives become active. Use `1` |
| `battlePassiveList[].passiveIDList` | Array of passive IDs — can mix base-game IDs and custom IDs |
| `supporterPassiveList[].level` | Awakening level for the supporter (ally) passive. Use `3` |
| `supporterPassiveList[].passiveIDList` | Typically one ID, paired with `CheckAwakenLevel3` in the passive definition |

### Using base-game passive IDs directly

The `passiveIDList` can reference base-game passive IDs verbatim. The game will apply those passives to the player identity exactly as it does to enemies.

For SojiAbiIdentity, the encounter's passive set (`soji.json → passiveSet.passiveIdList`) was used as-is:

| ID | Name | Effect |
|----|------|--------|
| `132219` | 천살성상[天殺星傷] | Combat start: TimeEntangleUnstable +4 |
| `132220` | (LittleFingerBoss_Shin source) | Grants 신(心)-지혜성 |
| `132222` | 새벽에서 황혼까지 | SP ≥ 0 → SojiAbiAgeFuture; SP < 0 → SojiAbiAgePast |
| `132223` | 시공간 잔상 분열 | SP delta each turn → adjusts TimeEntangleUnstable |
| `132214` | 미래 잔상 | Periodic forced skill mechanic |
| `132230` | 패닉 회복 | On panic end, restore SP to 30 |

> ⚠️ **Some base-game enemy passives reference boss-only buffs.**
> `132219` also grants `LvDownLittleFingerBossTwo` — a debuff that exists only in the boss context and will do nothing on a player (not a crash, just ignored).
> Test each ID you import and verify it doesn't destabilize the encounter.

---

## 3.3 `sojiabi_id_passive.json` — Passive Effect Definitions

```json
{
  "list": [
    {
      "id": 6821901,
      "attributeResonanceCondition": [
        { "type": "VIOLET", "value": 4 }
      ],
      "requireIDList": [
        "Modular/TIMING:AfterSlots/LUA:SojiIDPassive/LUAMAIN:onSojiPassive"
      ]
    },
    {
      "id": 6821921,
      "attributeStockCondition": [
        { "type": "VIOLET", "value": 4 }
      ],
      "requireIDList": [
        "CheckAwakenLevel3"
      ]
    }
  ]
}
```

### Activation condition fields

| Field | Used for | Meaning |
|-------|----------|---------|
| `attributeResonanceCondition` | Battle passive | Requires N resonance dice of the given sin on the team |
| `attributeStockCondition` | Supporter passive | Requires N total sin attribute dice on the team |

Both follow the same structure:
```json
{ "type": "VIOLET", "value": 4 }
```
This means "requires 4 Violet sin dice in the current team composition."

### `requireIDList` — what the passive actually does

This array can contain:
- **Built-in script IDs** like `"CheckAwakenLevel3"` — handled entirely by the engine
- **Modular trigger strings** — custom Lua-driven effects

A Modular trigger string format:
```
Modular/TIMING:<timing>/LUA:<lua_filename>/LUAMAIN:<function_name>
```

| Segment | Example | Meaning |
|---------|---------|---------|
| `TIMING` | `AfterSlots` | When to fire |
| `LUA` | `SojiIDPassive` | Lua file name (without `.lua`) in `modular_lua/` |
| `LUAMAIN` | `onSojiPassive` | Function to call inside that file |

---

## 3.4 Timing Options

| Timing | When it fires | Verified for identity |
|--------|--------------|----------------------|
| `AfterSlots` | After slot assignment, before skills execute | ✅ Confirmed (encounter + identity) |
| `RoundStart` | At the beginning of each round | ✅ Used in boss mods — may not fire for identity passives |
| `BeforeUse` | Before a specific skill fires | ✅ Used in skill `abilityScriptList` |
| `ChangeMotion` | During skill animation | ✅ Used in coin scripts |
| `EndBattle` | When battle ends | ✅ Confirmed in boss mods |

> ⚠️ **`TIMING:RoundStart` is unreliable for identity battle passives.**
> During testing of SojiAbiIdentity, `RoundStart` produced no visible effect while `AfterSlots` fired correctly. Use `AfterSlots` for identity Lua passives until `RoundStart` behavior is confirmed.

---

## 3.5 Lua Passive — `SojiIDPassive.lua`

```lua
-- SojiIDPassive.lua
-- 소지 아비 정체성 보조 패시브 (6821901)
-- 트리거: Modular/TIMING:AfterSlots/LUA:SojiIDPassive/LUAMAIN:onSojiPassive

function onSojiPassive()
    local r = round()

    if r == 1 then
        -- 전투 시작 시 신(心)-지혜성 최대치 보장 (base-game passive 132220 보완)
        local shinStack = bufcheck("Self", "LittleFingerBoss_Shin", "stack") or 0
        if shinStack < 3 then
            buff("Self", "LittleFingerBoss_Shin", 3 - shinStack, 99, 0)
            log("[SojiIDPassive] LittleFingerBoss_Shin +" .. (3 - shinStack), 0)
        end
    end
end
```

### Lua API reference (confirmed working in Lethe)

| Function | Signature | Returns |
|----------|-----------|---------|
| `round()` | `round()` | Current round number (int) |
| `bufcheck()` | `bufcheck(target, keyword, field)` | Stack/turn value (int), or nil |
| `buff()` | `buff(target, keyword, stack, turn, activeRound)` | — |
| `log()` | `log(message, level)` | — |
| `random()` | `random(min, max)` | Random int |
| `skillslotreplace()` | `skillslotreplace(slotIdx, fromID, toID)` | — (encounter only) |

**`buff()` parameter meanings:**

| Parameter | Meaning |
|-----------|---------|
| `target` | `"Self"`, `"Target"`, `"EnemyNoCores99"`, etc. |
| `keyword` | Buff ID string (e.g. `"Breath"`, `"LittleFingerBoss_Shin"`) |
| `stack` | Number of stacks to add |
| `turn` | Turns the buff lasts. Use `99` for "effectively permanent within one encounter" |
| `activeRound` | Delay before activation. `0` = immediate |

> ⚠️ **`turn: 0` behavior is buff-type dependent.**
> For some buffs `turn: 0` means permanent. For others it expires in the same frame.
> Safest default: use `turn: 99` for buffs you want to persist, `turn: 1` for single-round effects.

---

## 3.6 Supporter Passive

```json
{
  "id": 6821921,
  "attributeStockCondition": [
    { "type": "VIOLET", "value": 4 }
  ],
  "requireIDList": [
    "CheckAwakenLevel3"
  ]
}
```

`CheckAwakenLevel3` is a built-in engine check — it ensures the passive only applies when the supporter unit is at awakening level 3. Combined with `supporterPassiveList[].level: 3` in `pp.json`, this is the standard pattern for all supporter passives.

In `pp.json`:
```json
"supporterPassiveList": [
  {
    "level": 3,
    "passiveIDList": [6821921]
  }
]
```

> Supporter passives require `attributeStockCondition`, not `attributeResonanceCondition`.
> Using the wrong condition type causes the passive to never activate.

---

## 3.7 Passive Locale — `passiveList`

```json
{
  "6821901": {
    "a": 0,
    "id": "6821901",
    "name": "천살성상[天殺星傷]",
    "desc": "전투 시작 시, [폭주 - 잔상 얽힘] 4, [신(心) - 지혜성] 3 얻음\n매 라운드 시작 시, [순행 - 연민] 2 얻음 (최대 20)",
    "summary": "전투 시작 시 폭주-잔상 얽힘 4 + 신(心)-지혜성 3"
  },
  "6821921": {
    "a": 0,
    "id": "6821921",
    "name": "천살성상[天殺星傷]",
    "desc": "전투 시작 시, [폭주 - 잔상 얽힘] 4, [신(心) - 지혜성] 3 얻음\n매 라운드 시작 시, [순행 - 연민] 2 얻음 (최대 20)",
    "summary": "전투 시작 시 폭주-잔상 얽힘 4 + 신(心)-지혜성 3"
  }
}
```

The locale folder must be created manually if it doesn't exist:
```
custom_limbus_locale/KR/passiveList/sojiabi_id_passive_locale.json
```

Without this file the passive name appears as a raw ID string in the passive menu.

---

## 3.8 Base-Game Buff IDs Referenced

When using base-game buffs in Lua or skill scripts, use the internal buff keyword exactly as it appears in `dumpedData/limbus_locale/KR/bufList.json`.

| Buff keyword | Display name | Effect summary |
|--------------|-------------|----------------|
| `Breath` | 호흡 | Stacking count buff; consumed by certain skill effects |
| `Vibration` | 진동 | Stacking potency buff; detonated by VibrationExplosion |
| `PhantomIncision` | 잔상베기 | Stacking damage count buff |
| `TimeEntangleUnstable` | 폭주 - 잔상 얽힘 | On-hit: slash damage × stack count; 2+ burn, 3+ bleed, 4+ bind, 5+ poison |
| `LittleFingerBoss_Shin` | 신(心) - 지혜성 | Turn start: Breath power +5 and count +1 per stack; lost HP 15% → slash dmg +1 |
| `SojiAbiAgeFuture` | 순행 - 연민 | Per stack: dealt dmg −2%, received dmg −2% (max 20 stacks) |
| `SojiAbiAgePast` | 역행 - 미움 | Per stack: dealt dmg +2%, received dmg +2% (max 20 stacks) |
| `Haste` | 신속 | Speed +1 per stack |
| `OffenseLevelBuff` | 공격 레벨 증가 | Offense level +1 per stack |

---

## Summary — Encounter vs Identity Checklist

| | Encounter (`abnormality-unit`) | Identity (`personality`) |
|-|-------------------------------|--------------------------|
| Unit file | `abnormality-unit/*.json` | `personality/*.json` |
| Passive wiring | `passiveSet.passiveIdList` in unit JSON | Separate `personality-passive/*.json` |
| Passive condition | `attributeStockCondition` or none | `attributeResonanceCondition` (battle) / `attributeStockCondition` (supporter) |
| Coin flip fields | Not needed (always heads) | `color: "GOLD", grade: 1, prob: 0.5` required |
| `attributeType` | All types including CRIMSON | CRIMSON forbidden → crash |
| `skillTargetType: RANDOM` | Works as expected | May behave differently for players; prefer FRONT |
| Lua `skillslotreplace` | Works | No slot pattern — skill slots are player-controlled |
| Timing `RoundStart` | Confirmed working | Unreliable for identity passives; use `AfterSlots` |
| `appearance` source | Can use any enemy asset | Must use a personality asset string from `dumpedData/personality/` |
