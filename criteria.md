# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

**Two are written for you. You write three.**

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:** My search is a plain keyword-overlap match with no
synonyms and no stemming — it scores a listing by how many of the query's
exact words show up in its title, description, or style tags. A phrasing that
describes a real listing in different words than the listing itself uses (e.g.
asking for a "jean jacket" when the listing says "denim jacket") can score zero
and come back empty even though a matching item exists. That's a search miss,
not a loop bug, so I'm not claiming 5 of 5 for something my scoring method
can't guarantee.

<!-- Why 4 of 5 and not 5 of 5? Something about your search, probably —
     "my search is a plain keyword match and some phrasings will miss" is a
     real answer. -->

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:** This path is pure arithmetic, not fuzzy matching — the
price filter is a `<=` comparison and the size filter is an exact token
match, both over a fixed 40-item dataset. A query built to fail every filter
(e.g. a price ceiling below the cheapest listing) returns `[]`
deterministically every time, and the branch in `agent.py::run_agent` is a
plain `if not session["search_results"]`, no model call involved. Nothing here
is probabilistic the way keyword scoring or a model response is, so unlike
criterion 1, there's no reason to budget for a miss.

<!-- Why is 5 of 5 reasonable here when criterion 1 isn't? What's different
     about this path? -->

---

## 3. Something about state

Given a matching query, `session["selected_item"]["id"]` equals
`session["search_results"][0]["id"]`, and that same `id` is the one found on
the dict `suggest_outfit` actually receives as its `new_item` argument — in 5
of 5 tries.

<!-- YOU WRITE THIS ONE.

     How would you know that the item your search found is the same item the
     next tool received? Name something countable or observable.

     This is the criterion people find hardest, because state failure doesn't
     look like state failure — it looks like a tool problem. Something that
     compares session["selected_item"] against what actually reached
     suggest_outfit is the shape you're after. -->

**Why this target:** Moving a dict through the session is plain in-process
assignment — no network call, no randomness, nothing that varies run to run.
If `run_agent` ever passed the wrong item (a stale reference, a copy that got
mutated, an off-by-one on which search result got chosen), it would be wrong
every single time the same code path ran, not intermittently. So 5 of 5 is the
only honest target: anything less would mean "the state bug only happens
sometimes," which isn't how a wiring mistake behaves. I verified this by
wrapping `suggest_outfit` with a spy that records the id of the object it was
called with, then comparing that to `session["selected_item"]["id"]` after the
run — see the Sample Run section.



---

## 4. Something about the fit card

For 5 different matching items, each run with `CACHE_ENABLED` off, every fit
card mentions its item's price and platform at least once, and no two of the
five cards share an identical opening sentence — in at least 4 of 5 tries.

<!-- YOU WRITE THIS ONE.

     The fit card calls a model, so the same input can produce different words
     each time. That's not a bug — it's the nature of the tool. So what would
     make it acceptable?

     Think about what you'd actually be unhappy to see. A caption that never
     mentions the price? Two different items producing the same opening
     sentence? A card longer than a caption anyone would post? Any of those can
     be turned into a number. -->

**Why this target:** I care about two failure modes: a caption that's
generic enough to fit any item (same opening sentence every time, which would
mean the prompt isn't actually using the item's details), and a caption that
drops the price or platform entirely (which would make it useless as a listing
caption). Both are checkable without a human reading the prose. I set 4 of 5,
not 5 of 5, because `create_fit_card`'s prompt *asks* the model to mention
price and platform once each but can't force it — a generative instruction is
a request, not a guarantee, and I've seen models occasionally paraphrase a
price away (e.g. "under $40" instead of restating "$38.00"). One miss in five
is the model being a model; more than that would point at my prompt.



---

## 5. Your choice

Given a `max_price` ceiling, 100% of the listing dicts `search_listings`
returns have `price <= max_price` — checked across 5 different ceilings run
against the full 40-listing dataset.

<!-- YOU WRITE THIS ONE TOO.

     Pick something you actually care about getting right. Speed, the empty
     wardrobe path, what happens when the model can't be reached, whether the
     search respects a price ceiling — anything, as long as it names a number
     or an observable outcome. -->

**Why this target:** A price ceiling is the one filter in this app that's a
hard promise, not a preference — if someone asks for "under $30" and gets back
a $45 jacket, that's not a weak match, it's a broken filter, and I'd rather
catch that than hide it in a 4-of-5 target. The check is a plain `<=` over
data that doesn't change between runs, so unlike keyword scoring there's no
legitimate reason for it to be right only some of the time. 100% is the target
because anything short of it means the filter is wrong, not just imprecise.



---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
