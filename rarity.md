# DNS Name Rarity (ÐNS)

**Version:** rarity-v1  
**Status:** Launch Draft  
**Applies to:** product + indexers (deterministic overlay — not an on-chain field)  
**Familiar UX:** OpenSea / Magic Eden traits · Rare Koinu tier names

---

## 1. Why

Bitcoin SNS taught the world that short names matter, then stopped at informal clubs.
ÐNS publishes a **trait → score → tier → clubs** system anyone can recompute from the
name string alone. Rarity is structural scarcity, not vibes. Markets still price taste.

Goals:

- Fair enough that users worldwide can agree on the *method*
- Familiar to NFT traders (trait grid, score, tier badge, club chips)
- Works on **unminted** names (show rarity before register — the SNS gut-punch)
- Frozen wordlists (versioned). No live APIs. No AI coolness scores.

Non-goals:

- Claiming rarity equals floor price
- Culture-ranking which first name is “better”
- Paying for a higher tier
- Mixing mint height into rarity (show provenance separately)

---

## 2. Pipeline

```text
name → normalize → traits → intrinsic score → hard-gate tier → clubs
```

Optional later: empirical `#rank / supply` within a namespace from indexer histograms.
v1 ships **intrinsic** score + tier so available names already show coveted badges.

---

## 3. Traits

Score the **label** (left of `.`). Namespace is its own trait.

| Trait | Values |
| --- | --- |
| `lengthBucket` | `1` · `2` · `3` · `4` · `5` · `6` · `7+` (grapheme length) |
| `charset` | `letters` · `digits` · `mixed` · `emoji` · `hyphen` |
| `pattern` | `single` · `double` · `triple` · `palindrome` · `sequential` · `repeating` · `none` |
| `wordClass` | `first-name` · `dictionary` · `doge-meme` · `numeric-club` · `other` |
| `namespaceClass` | `hero` (`.doge`) · `flagship` · `curated` · `open` |

### 3.1 Charset rules

- ASCII digits only → `digits`
- ASCII letters only → `letters`
- ASCII letters + mid hyphen → `hyphen`
- ASCII letters + digits (and optional hyphen) → `mixed`
- Any non-ASCII scalar (emoji / other scripts) → `emoji`

### 3.2 Pattern rules (first match wins)

1. grapheme length 1 → `single`
2. all graphemes identical, len 2 → `double`; len ≥ 3 → `triple`
3. grapheme palindrome, len ≥ 2 → `palindrome`
4. ASCII sequential run (`123`/`321`/`abc`/`cba`) → `sequential`
5. simple 2-grapheme repeat (`abab`, `1212`) → `repeating`
6. else → `none`

### 3.3 Word class (first match wins)

1. Pure digits with numeric value ≤ 9999 → `numeric-club`
2. Label in **first-names-v1** → `first-name`
3. Label in **doge-meme-v1** → `doge-meme`
4. Label in **dictionary-v1** → `dictionary`
5. else → `other`

Wordlists are frozen under product `lib/dns/wordlists/v1/` (and mirrored in docs).
A list bump is a new rarity version (`rarity-v2`), never a silent reshuffle.

### 3.4 Namespace class

- `doge` → `hero`
- `dogecoin` · `doginals` → `flagship`
- Other curated studio suffixes → `curated`
- Everything else → `open`

---

## 4. Intrinsic score (rarity-v1 weights)

Higher weight = rarer contribution. Score = sum of weights for the five traits.

| Trait value | Weight |
| --- | ---: |
| length `1` | 1000 |
| length `2` | 400 |
| length `3` | 120 |
| length `4` | 40 |
| length `5` | 12 |
| length `6` | 4 |
| length `7+` | 1 |
| charset `emoji` | 80 |
| charset `digits` | 15 |
| charset `hyphen` | 10 |
| charset `letters` | 8 |
| charset `mixed` | 5 |
| pattern `single` | 200 |
| pattern `triple` | 90 |
| pattern `double` | 60 |
| pattern `palindrome` | 35 |
| pattern `sequential` | 25 |
| pattern `repeating` | 20 |
| pattern `none` | 1 |
| word `first-name` | 25 |
| word `doge-meme` | 18 |
| word `dictionary` | 12 |
| word `numeric-club` | 8 |
| word `other` | 1 |
| ns `hero` | 20 |
| ns `flagship` | 10 |
| ns `curated` | 4 |
| ns `open` | 1 |

---

## 5. Tier ladder (Rare Koinu names)

```text
mythic → legendary → epic → rare → uncommon → common
```

### 5.1 Hard gates (any match → at least that tier)

| Tier | Gate |
| --- | --- |
| **mythic** | length 1 |
| **legendary** | length 2 |
| **epic** | length-3 letters · OR numeric value ≤ 999 · OR palindrome with length ≤ 4 |
| **rare** | length-4 letters · OR numeric value ≤ 9999 · OR first-name with length ≤ 5 |
| **uncommon** | length 5 · OR dictionary/doge-meme · OR pattern ∈ {double,triple,sequential,repeating} |
| **common** | else |

### 5.2 Score floors (raise tier only — never lower a hard gate)

| Tier | Minimum score |
| --- | ---: |
| mythic | 1100 |
| legendary | 500 |
| epic | 160 |
| rare | 70 |
| uncommon | 30 |
| common | 0 |

Final tier = max(hard-gate tier, score-floor tier).

---

## 6. Clubs (display chips)

Emit zero or more:

- `Single` · `2L` · `3L` · `4L`
- `999` (numeric ≤ 999) · `10k` (numeric ≤ 9999)
- `Letters` · `Digits` · `Emoji` · `Hyphen`
- `Palindrome` · `Double` · `Triple` · `Sequential`
- `First Name` · `Dictionary` · `Doge Meme`
- `.doge` when namespace is hero

Clubs are tags for filters and flex — they do not replace tier.

---

## 7. Product UX

Name profile (`/names/:name`) SHOULD show:

- Tier badge
- Intrinsic score + rarity version
- Trait grid (OpenSea-style)
- Club chips

Explore / market MAY filter by club and sort by score.
Empirical `#rank / N` MAY be added when dogex exposes namespace histograms; until then
do not invent fake ranks.

---

## 8. Indexer note

Rarity is a **deterministic overlay**. Indexers MAY attach `rarity` on DNS JSON
responses. They MUST use the same versioned rules. Omitting rarity is valid; clients
MAY compute locally from the name string.
