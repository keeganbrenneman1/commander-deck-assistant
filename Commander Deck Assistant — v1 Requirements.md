# Commander Deck Assistant — v1 Requirements

Oct 10, 2026 · @Keegan Brenneman

## Purpose and scope

v1 helps an invested player tune an existing Commander deck, starting with a tuned copy of Dance of the Elements, the five-color Elementals precon led by Ashling, the Limitless. Building and tuning are one product with different entry points; v1 ships only the "existing list" entry point.

**v1 persona.** A player who owns a precon, has already made some changes, and wants to know whether those changes worked and what to change next. The newcomer described in the brief's Why is served by later stages.

**v1 user.** v1 has two users, the PM and his brother, sharing one passcode and one space where both see every deck, with no owner field. The PM's judgment still defines "better" for now. Testing with other players comes later.

**Format.** Magic: The Gathering, Commander only.

**In scope.** Six stories: import, classify, tune, tier evaluation, budget and compare, plus editing and managing versions. Everything else is listed under Out of scope.

## Core concepts and data model

Every change creates a new deck version, and every version carries the goal it was trying to reach. That pairing is what lets the app judge whether a change was good.

| Concept | What it is | Key fields |
| --- | --- | --- |
| Deck | A commander and its history of lists | Name, commander(s), color identity, source precon (optional) |
| Version | A saved snapshot of the 100-card list | Version number, name, cards, parent version, saved date, iteration goal |
| Iteration goal | What this version set out to improve | Ranked axes (power, consistency, fun, trolly), must-include cards, synergies, per-card price cap in EUR, target deck total in EUR (optional), target tier (optional), note |
| Card | A reference to real card data | Scryfall ID, name, mana value, type line, oracle text, color identity, EUR price, Game Changer flag |
| Function tag | The role a card plays in this deck | Tag, source (app or user) |
| Suggestion | A proposed change | Card in, card out, reason, axis served, price, effect on tags and tier |

Three rules apply across the model:

- **Versions are immutable.** A saved card list never changes; edits always produce a new version. A version's name and goal can be edited later.
- **The precon is version 0.** When the source precon is known, it becomes the baseline for comparisons.
- **Function tags belong to the deck, not the card.** The same card can be a threat in one deck and a role player in another. User corrections carry forward to later versions of the same deck.

## Functional requirements

Requirements are grouped by user story. Each has an ID so the flow, the build brief and test cases can refer to it.

### Import

- **IMP-1** Accept a pasted plain-text decklist in the common "1 Card Name" format that Moxfield, Archidekt and most tools export. Plain text avoids building one integration per site.
- **IMP-2** Match every card name against Scryfall. Unmatched names are listed for the user to fix; the app never guesses.
- **IMP-3** Identify the commander(s). If it's ambiguous, ask the user.
- **IMP-4** Check Commander legality: 100 cards, one copy of each card except basic lands, color identity and the banned list. Show problems as warnings; the user can still save.
- **IMP-5** Optionally link the source precon as version 0, either from a precon dataset (pending verification) or a second pasted list.
- **IMP-6** Save the import as a new version.

### Classify

- **CLS-1** Automatically tag every card with all the function tags that apply from a fixed v1 list: ramp, card draw, targeted removal, board wipe, protection, tutor, threat, engine, finisher, combo piece, utility and land. A card can carry several tags.
- **CLS-2** Classify from the card's oracle text in Scryfall data, never from its name alone.
- **CLS-3** Show whether each tag came from the app or the user.
- **CLS-4** Let the user add, remove or change tags. User tags always override app tags and carry forward to later versions.

### Tune

- **TUN-1** Before suggesting anything, ask for an iteration goal: ranked axes, must-include cards, synergies, a budget and an optional target tier.
- **TUN-2** Build candidates from real data first: query Scryfall for legal cards in the deck's color identity that fit the goal. The LLM then ranks and explains from that candidate set only.
- **TUN-3** Return ranked add/cut suggestions, 5 to 15 by default. Each shows card in, card out, the reason, the axis it serves, its price, and its effect on function counts and tier.
- **TUN-4** Before display, drop any suggestion naming a card that isn't in Scryfall data, isn't legal for the deck, or breaks the goal's constraints.
- **TUN-5** Score power and consistency with measured numbers. Treat fun and trolly as guidance that shapes suggestions, and say in the UI that they aren't scored.
- **TUN-6** Never suggest cutting a must-include card.
- **TUN-7** Warn when a suggested cut removes an engine or synergy piece tied to the goal. Never silently filter out stronger options.
- **TUN-8** Let the user accept or reject each suggestion. Accepted changes form a draft that becomes a new version when saved.

### Tier evaluation

- **TIER-1** Show an estimated Commander Bracket (1 to 5) for each version.
- **TIER-2** Separate the hard checks the app can count, such as Game Changers and two-card combos, from the parts that need judgment. Label which is which.
- **TIER-3** List the cards that set the deck's minimum bracket and explain why.
- **TIER-4** Pull the Game Changers list from live data rather than hardcoding it, and show the date the bracket rules were last updated.

### Budget

- **BUD-1** Set the budget in euros as a maximum price per card for the cards added in one iteration. This cap is the hard constraint on suggestions.
- **BUD-2** Use Scryfall's EUR (Cardmarket) prices and show when each price was last updated.
- **BUD-3** Never offer a suggested add priced above the per-card cap. Separately, show the deck's total value against an optional target total. The target is a readout, not a filter, so cards can be collected over time.
- **BUD-4** Show cards with no EUR price with a "price unknown" flag rather than hiding them.

### Compare

- **CMP-1** Compare any two versions. Default to the current and previous version, with one tap to compare against version 0.
- **CMP-2** Show the mana curve of both versions.
- **CMP-3** Show counts by printed type: lands, artifacts, enchantments, creatures, planeswalkers, instants and sorceries.
- **CMP-4** Show counts by function tag. Because a card can carry several tags, these counts can add up to more than 100.
- **CMP-5** Show each version's estimated bracket and Game Changers count.
- **CMP-6** List the cards added and removed.
- **CMP-7** Show the iteration goal beside the results for the axes it chose, with a short plain-language summary. The summary may only refer to numbers the app computed.

* **CMP-8** Show color splits for both versions: colored mana symbols by color and mana sources by color. This matters most in a five-color deck.

### Versions

- **VER-1** Edit a version's list by hand: add or remove cards by name (matched against Scryfall) and set basic land counts. Edits form a draft that becomes a new version when saved.
- **VER-2** Name and rename versions.
- **VER-3** Delete a version only if no other version was built from it. Version 0 can never be deleted.
- **VER-4** Add or change the goal on an existing version, including lists built before the app existed. New suggestion requests still require a goal first (TUN-1); manual edits don't.

## Flow

The product is a loop, not a line. Importing and tagging happen once; every iteration after that starts with a goal and ends with a comparison against it.

&#91;embedded content: tuning loop · import once, then repeat\]

1. **Import.** Reference the precon as version 0, or paste a list (IMP).
2. **Review tags.** The app tags every card by function; the user corrects anything it got wrong (CLS).
3. **See the deck.** Curve, color splits, card types, function counts, tier and deck value (CMP, TIER, BUD).
4. **Set a goal.** Choose axes, must-include cards, synergies, a per-card price cap and an optional target tier. A goal can also be added to an existing version, such as a list tuned before the app existed (TUN-1, VER-4).
5. **Get suggestions.** Ranked adds and cuts within the price cap; accept or reject each one (TUN, BUD).
6. **Edit by hand.** Add or remove cards and adjust basic land counts (VER-1).
7. **Save a version.** Name it; the saved list never changes (VER-1, VER-2).
8. **Compare.** Against the previous version or version 0, measured against the goal. Then back to step 4 (CMP, TIER).

Versions can be renamed at any time, and deleted only when nothing else was built from them (VER-2, VER-3).

## Grounding and data sources

Real data supplies every fact; the LLM only ranks, classifies and explains. Two sources still need verification before they become hard requirements.

| Data | Source | Status |
| --- | --- | --- |
| Cards, oracle text, legality, color identity | Scryfall API and bulk data (free, no key) | Confirmed |
| Prices in EUR | Scryfall's Cardmarket price field | Confirmed |
| Game Changers list | Scryfall search for Game Changers | Confirmed |
| Bracket criteria | [Wizards' Commander Brackets announcements](https://magic.wizards.com/en/news/announcements/commander-brackets-beta-update-february-9-2026), stored as a dated config | Confirmed; updated by hand |
| Precon decklists | MTGJSON | To verify |
| Two-card combos | Commander Spellbook | Confirmed |

Rules for the LLM:

- **No invented cards.** The LLM may only name cards from the candidate set the app retrieved.
- **No invented facts.** Anything stated about a card (cost, text, legality, price) comes from its Scryfall record.
- **Validate before display.** LLM output is checked against card data, and failures are dropped, not shown.
- **One swappable seam.** Card data retrieval sits behind a single function so mock data and live data can be swapped without touching the rest of the app.

The bracket system is officially still in beta and changes several times a year, so bracket rules are treated as data with an as-of date, not as code.

## Non-functional requirements

The app must be buildable, deployable and usable entirely from an iPad, at as close to zero cost as possible.

- **NFR-1 iPad-only development.** No step may require a laptop or local dev environment. Build through Claude Code and browser-based tools.
- **NFR-2 Free-tier hosting and storage.** Hosting and database must fit free tiers at single-user volume.
- **NFR-3 LLM spend only where judgment is needed.** Call the LLM for classifying, ranking suggestions and writing summaries. Counting, curves, legality and bracket checks run in plain code.
- **NFR-4 Cache classifications.** Store each card's app-generated tags by card so re-imports and later versions don't pay to classify the same card twice.
- **NFR-5 Works in iPad Safari.** Touch-friendly controls, readable in portrait and landscape, no hover-only interactions.
- **NFR-6 Persistence.** Decks, versions, goals and tag corrections survive between sessions.
- **NFR-7 Freshness is visible.** Prices and bracket rules always show their as-of dates.
- **NFR-8 Respectful API use.** Stay within Scryfall's published rate limits; prefer bulk data and caching over repeated live calls.

* **NFR-9 Passcode gate.** With no login, the app sits behind a single passcode stored as a Vercel environment variable, and the Anthropic console has a hard spend limit of €5 per month. Nobody can run paid LLM calls just by finding the URL.

Stack: Next.js with TypeScript, Supabase and Vercel, the same as the Job Application Assistant, chosen because all three have free tiers and are already proven from an iPad.

## Success criteria

v1 succeeds when the Elementals deck can be taken through one full iteration: imported, compared against the precon, tuned against a goal, and saved as a new version that passes every check below.

| Success criterion | Requirements | Passes when |
| --- | --- | --- |
| Evolve the deck within a budget | TUN-1, BUD-1 to BUD-3 | No added card exceeds the per-card cap, and the deck's total value shows against its target |
| Include certain cards | TUN-1, TUN-6 | Every must-include card is in the saved version |
| Include certain synergies | TUN-1, TUN-7 | No synergy piece tied to the goal was cut without a warning |
| See how the curve changes | CMP-2 | Curves for both versions display side by side |
| See how card type counts change | CMP-3 | Printed type counts and differences display for both versions |
| See how function counts change | CLS-1 to CLS-4, CMP-4 | Function counts display, including user corrections |
| Identify better cards using the tiers | TIER-1 to TIER-4, TUN-3 | Each suggestion shows its tier effect, and the version's bracket and the cards behind it are shown |
| Judge whether past changes helped (milestone 1) | IMP-5, CMP-1, CMP-7 | The tuned list can be compared against the precon as version 0 |

## Out of scope and open questions

**Out of scope for v1:** building from favorite cards, colors or strategies (v2); budget-first deck recommendations; the simulator; links to buy cards; collection tracking; formats other than Commander; other TCGs; sharing and multiple users; the new-player questionnaire.

**Open questions:**

- [x] Does MTGJSON include the Dance of the Elements precon list (Lorwyn Eclipsed, January 2026)? To check in the data spike session. If not, IMP-5 falls back to pasting it.
- [x] Commander Spellbook has a public, open-source API: one endpoint finds combos in a decklist, including combos one card away, and another estimates the bracket. Resolved: the project is MIT-licensed, has a TypeScript client, and its lead backend developer has publicly invited apps to use the bracket endpoint. No rate limit is published, so the app calls it only when a version is saved and caches the result, and credits Commander Spellbook in the UI. If unusable, TIER-2 drops automatic combo checks.
- [x] Is the function tag list in CLS-1 the right one for v1? Resolved: merged list locked in CLS-1, with multiple tags per card.
- [x] Does persistence need a login, or is a single unauthenticated user acceptable for v1? Resolved: we do not need a login at this time, but will use a single passcode
- [x] Resolved: LLM spend is capped at €5 per month across both users, enforced by the Anthropic console spend limit. Manual edits, comparisons and tier checks run in plain code and cost nothing.
