---
name: short-trip-planning
description: "Use when planning a weekend or short family trip."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [travel, trip-planning, family, weekend, accommodation]
    category: productivity
---

# Short-Trip Planning

Plan weekend/short family trips end to end: find candidate accommodations
(farms, guesthouses, hotels), filter by real driving distance from home,
pull actual prices from each provider's own site, and surface booking
constraints (minimum stay!) before recommending anything.

## Workflow (validated order matters)

1. **Resolve dates first.** If the user says "this weekend", compute the
   concrete Fri–Sun dates from the current date immediately (via `date`) and
   plan for THOSE dates — do not plan generically or ask which weekend. All
   later checks (minimum stay, availability, season pricing) hang on exact
   dates. (Correction from session 08/2026: I planned a generic "Freitag
   bis Sonntag" trip and was corrected mid-run with "Es geht um
   dieses Wochenende".)
2. **Find named candidates via REGIONAL/OFFICIAL portals, not generic
   web searches.** Generic queries ("Bauernhofurlaub Familie") return blog
   noise; regional tourism association portals list real named businesses.
   Germany examples: landorado.de (BW farm stays), landsichten.de,
   bauernhofurlaub.de, plus region portals like naturerlebnis-hayingen.de.
   Details and pricing patterns: references/farm-stay-sources.md.
3. **Distance-filter with the maps skill.** Geocode home once, then run
   `distance "home" --to "<each candidate town>"` in ONE terminal loop.
   Present real road distance + duration (straight-line lies in rural
   areas). Drop anything over the user's radius before deep-diving.
   NOTE: get the maps script path from `skill_view(name='maps')`
   (its `skill_dir` field) — documented paths have been stale before.
4. **Pull prices and constraints from each provider's OWN website** (found
   via targeted `web_search "<Hofname> <Ort> Preise"` then `web_extract`).
   Portal listings often hide prices; the provider site has them.
5. **Check the minimum-stay rule BEFORE recommending.** This is the #1
   weekend-trip killer: many farm stays rent weekly only (some 14-night
   minimum in summer), others charge a Kurzaufenthaltszuschlag (short-stay
   surcharge, e.g. +80 €) or price per person/night. A perfect farm that
   refuses 2-night stays is not a recommendation for a weekend trip —
   flag it honestly and prefer weekend-friendly hosts (group farms,
   pensions) or note phone-negotiable exceptions.
6. **Present a shortlist** with: distance/drive time, price for the exact
   group size and nights (do the math — surcharges, extra-person fees,
   child age tiers), booking constraint, and contact (phone matters — for
   short-notice weekends, calling often unlocks exceptions to the
   published minimum stay).

## Presentation preferences (Michael, Telegram, German)

- Respond in German. Grouped recommendations with ⭐ top pick, bullet
  facts: 🚗 km/Fahrzeit, price per unit/night and total for the trip.
- Always give a concrete total price for the user's actual group size
  (adults + children with age tiers), not just the base nightly rate.
- State honestly when a listing's minimum stay conflicts with the ask,
  and offer the workaround (call and ask for an exception, or the
  weekend-friendly alternative nearby).

## Pitfalls

- Per-unit vs per-person pricing: group farms often charge per
  person/night with child discounts — compute the family total.
- Season tables: summer = Hauptsaison for many farms; use the dates'
  actual season row, not the cheapest published price.
- Portal prices are lower bounds ("ab"-prices); provider sites or phone
  give the real number.
- Don't recommend a place as "available" without checking its booking
  calendar page when one exists (Belegungskalender).
