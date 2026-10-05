# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

A user types one plain-language line describing what they're shopping for —
something like `vintage graphic tee under $30` — and FitFindr turns that into a
structured search of a secondhand-listings dataset, picks the best match, and
hands back three things: the listing itself, an outfit built either from
pieces they already own or general styling advice if their wardrobe is empty,
and a short caption written like a real post about the find. If nothing in the
data matches what they asked for, it stops after the search and tells them
what to loosen — the size, the price ceiling, or the keywords — instead of
pushing ahead with nothing to work from.

---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Filters the listings data by size and price ceiling, then ranks what's left by keyword overlap with the description, returning the best matches first.
- **Inputs:** `description` (str), `size` (str or `None`), `max_price` (float or `None`)
- **Returns:** A list of listing dicts, best match first, at most `config.SEARCH_RESULT_LIMIT` of them — each with `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand` (str or `None`), `platform`. Score = number of description keywords (lowercased, split on whitespace, punctuation stripped) found anywhere in the listing's `title` + `description` + `style_tags`; ties break by lower price first. A size argument only matches a whole token of the listing's `size` field split on `/` and whitespace, case-insensitive — so `"M"` matches `"S/M"` but `"S"` never matches `"US 9"`.
- **When it has nothing:** Returns `[]` — an empty list, never `None`, never an exception.

### `suggest_outfit`

- **What it does:** Asks the model to pair a candidate item with one or two outfits, built from the user's wardrobe when they have one, or general styling advice when they don't.
- **Inputs:** `new_item` (dict — a listing dict), `wardrobe` (dict with key `items` → list of wardrobe item dicts, possibly empty)
- **Returns:** A non-empty plain-prose string, 2–4 sentences, no markdown bullets. When `wardrobe["items"]` is non-empty, it names specific wardrobe items by their `name`/`category` field. When it's empty, it gives general styling advice for the item instead — same length and shape of response either way.
- **When it has nothing:** There's no "nothing" case for the wardrobe — empty wardrobe is a valid input handled by the general-advice branch above, not an error. The only failure mode is `ModelUnavailable` bubbling up from `generate()`, which `suggest_outfit` does **not** catch — that's handled one level up, in `agent.py::run_agent`.

### `create_fit_card`

- **What it does:** Writes a short caption, in the voice of someone posting about their thrift find, that mentions the item, its price, its platform, and the suggested outfit.
- **Inputs:** `outfit` (str — the return value of `suggest_outfit`), `new_item` (dict — a listing dict)
- **Returns:** A string, 2–4 sentences, that mentions `new_item["title"]`, `new_item["price"]`, and `new_item["platform"]` each exactly once.
- **When it has nothing:** If `outfit` is empty or whitespace-only, returns the fixed string `f"No fit card available — no outfit suggestion to build one from for {new_item['title']}."` rather than calling the model or raising.

---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->

**Branch rule:** If `search_listings` returns an empty list, put a message in `session["error"]` naming what to change (loosen the size, raise the price ceiling, or try different keywords) and return the session — do not call `suggest_outfit` or `create_fit_card`. Otherwise, take `search_results[0]` as `selected_item` and go on to `suggest_outfit`, then `create_fit_card`.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** Regex — one pattern pulls a price ceiling out of phrases like "under $30" or "below $40", another pulls a size out of phrases like "size M"; whatever text is left after stripping both becomes the `description` passed to `search_listings`.

**What moves through the session:** `query` → `parsed` (`description`, `size`, `max_price`) → `search_results` → `selected_item` → `outfit_suggestion` → `fit_card`, in that order, with `error` set and the run stopped short if `search_results` comes back empty.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask 'vintage graphic tee under $30'

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   You can create a quintessential Y2K daytime look by pairing the butterfly baby tee with your baggy dark-wash straight-leg jeans and chunky white sneakers. For a slightly edgier vibe when the temperature drops, layer your black cropped zip hoodie right over the top and finish the outfit with your black combat boots.

  Fit card: Found this literal dream of a butterfly baby tee on depop for just $18.00 and I'm never taking it off. It gives major 2000s pop princess energy, especially paired with baggy dark wash denim and chunky sneakers. Honestly obsessed with how easy it is to throw on and instantly look the part.

0 model calls this session, 2 served from cache
```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
[{'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'price': 15.0, ...}, {'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'price': 18.0, ...}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'price': 19.0, ...}, {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'price': 24.0, ...}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'price': 27.0, ...}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'price': 26.0, ...}]
```

(dicts truncated here for readability — each one carries the full listing shape described in Tool Inventory above. Worth noting: `lst_017`, a mesh top, outranks the actual graphic tees because its *description* happens to mention "graphic tee" in passing — plain keyword overlap can't tell a real match from an incidental one. That's exactly the limitation Criterion 1 budgets for.)

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
You can create a classic streetwear look by pairing these vintage Levi's with your white ribbed tank top and black combat boots. Layer the black denim jacket over top and accessorize with your black crossbody bag for an effortless monochrome contrast. Alternatively, throw on your oversized grey crewneck sweatshirt with the medium wash jeans and chunky white sneakers for a relaxed, everyday outfit.
```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"
Finally tracked down these dream medium-wash 501s on depop for just $38.00, and the fit is absolute perfection. They've got that ultimate 90s slouchy vintage vibe that I've been hunting for forever. Can't wait to live in these with my favorite white sneakers all spring.
```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:* I described the size-matching problem the `search_listings` docstring warns about — that `"s" in "us 9"` is `True` and so is `"l" in "xl"` — and asked for a matching rule that wouldn't have that bug, given the actual size strings in the data (`"S/M"`, `"XL (oversized)"`, `"US 8.5"`, `"W30 L30"`).
- *What came back:* A rule that splits each listing's `size` field into whole tokens on `/` and whitespace, lowercases them, and checks for an exact token match — so `"M"` matches `"S/M"` (tokens `{"s", "m"}`) but never matches `"US 9"` (tokens `{"us", "9"}`).
- *What I changed:* I implemented `_size_tokens()` exactly that way in `tools.py`, and wrote the matching rule into the Tool Inventory's `search_listings` entry so the behavior is documented, not just implicit in the code.

**Moment 2**

- *What I asked for:* I ran `create_fit_card` on the same item three times in a row and got back three word-for-word identical captions, so I asked what in the starter would cause that.
- *What came back:* Two candidates, both in `config.py` — `CACHE_ENABLED` (identical prompts reuse a cached answer) and `TEMPERATURE` (at `0.0`, the model is deterministic). Since `TEMPERATURE` was already `0.9`, the suggestion was to isolate the cache by forcing it off for one run.
- *What I changed:* I reran the same three calls with `AI201_CACHE=0` and got three genuinely different captions back, which confirmed the cache — not the temperature — was the cause. I didn't change any code for this (the cache is working as designed), but it changed how I read "identical output" while testing: it's not automatically a sign `create_fit_card` is broken.

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
