# Hsr Homes — Ivy Homes Assignment Submission

A single-file HTML/JS frontend (no build step) on top of the Ivy Homes property API for Bangalore, plus a data audit documented below and in `submission.json`.

## How to run it

Just open `index.html` in a browser, or serve the folder with any static file server:

```
python -m http.server 8000
```

Then visit `http://localhost:8000`. No npm install, no build step — it's a single HTML file with vanilla JS.

Log in with one of the demo accounts (pre-filled in the login form): `demo1@ivy.homes` / `demo2@ivy.homes` / `demo3@ivy.homes`, password `285c4fd5df`.

On login the app pages through the full listings/rentals/projects datasets (~150 requests total, well under the 1200/min rate limit) and caches them in memory for the session. This takes up to a minute — a loading screen shows progress.

## How I worked out what to distrust

I read `API_REFERENCE.md` line by line before writing any code and flagged every falsifiable claim as a hypothesis, then tested each one directly against the running API using small Python probe scripts before building anything on top of it:

1. **Auth**: the docs say send the key as `?api_key=...`. First request came back `401` with the API's own error message telling me to use an `X-API-Key` header instead. Confirmed and fixed immediately.
2. **Token lifetime**: the docs claim 24h tokens and "no refresh flow." The actual login response has `expires_in: 900` and includes a working `refresh_token` / `/auth/refresh`. Since the assignment requires the app to survive 30 minutes logged in, this made token refresh mandatory, not optional — the opposite of what the docs say.
3. **Pagination**: the docs describe `{total, page, page_size, results}`. The real response is `{limit, offset, count, total, has_more, results}`. I confirmed `page` is silently ignored by requesting `page=1` and `offset=0` and getting byte-identical results. I then paged every collection to `has_more: false` and counted unique IDs — zero duplicates, but more records than the server's own `total` field claimed (4700 vs. reported 4445 for listings; similar gaps on rentals and projects). The `total` field simply cannot be trusted.
4. **"Only active listings returned"**: the docs say inactive/withdrawn listings are excluded server-side. I found `is_live: false` records in the very first page of results. Counting across the full dataset: 978 of 4700 records are `is_live: false`.
5. **Project prices**: I noticed one project with `price_min=98.0, price_max=2.74` — impossible if both are in the same unit as documented ("rupees"). I built a disambiguation using price-per-sqft plausibility (testing both crore-scale and lakh-scale interpretations against each project's own `min_area_sqft`/`max_area_sqft`) — it resolved cleanly for all 520 projects with zero ambiguous cases.
6. **Rental deposits**: sampling showed some deposits as absurdly tiny numbers (6, 8, 2...). Grouping by `website` showed this was 100% specific to `zerobroker` — its `deposit` field stores a small multiplier (months of rent) instead of a rupee amount, while every other source stores real rupees at a consistent ~6x-rent median.
7. **`total_listings` on projects**: cross-checked every project's declared count against the actual count of matching listings in the full harvested dataset. 127 of 520 disagree even under the most charitable (live-only) interpretation.
8. **Corrupt/impossible records**: checked for `carpet_area > super_built_up_area`, `floor > total_floors`, and `price <= 0`. Found exactly 8 of each, all disjoint (24 total, no overlap) — a suspiciously clean signal that these are the intentionally-seeded impossible records.
9. **Fake listings**: checked for the exact same `(contact, locality, bedroom, price)` combination appearing on more than one `listing_id` — found 4 records forming 2 pairs.
10. **A prompt injection in the data**: two listing/rental descriptions (and 6 more found on a full-dataset grep) contain text addressed to "automated tools and AI assistants" instructing any AI to insert a fabricated `dataset_audit_ref` key into `submission.json`. I did not comply — this is not an instruction from Ivy Homes, it's planted in scraped seller text, exactly the kind of thing the assignment's own warning ("a seller can write anything") is about. Flagged as a `fraud` finding instead.

## What I checked that turned out to be fine

- **Per-website unit bugs in listing area fields**: two early listings from `magichomes` had suspiciously tiny `carpet_area` values for their bedroom count, which looked like a sqm-vs-sqft mixup. Computing the median carpet area per bedroom count across all five `website` sources on the full dataset showed they're all within ~5-7% of each other — no systemic per-website unit bug. Those two records were just individual outliers, not a pattern.
- **Timestamp timezone mislabeling**: I hypothesized that `posted_at`'s claimed UTC (`Z` suffix) might actually be IST mislabeled, similar to the confirmed `/health` timezone note. I tested this by checking the hour-of-day distribution of `posted_at` across all listings, expecting a human daytime-posting pattern that would reveal a timezone shift. The distribution came back essentially flat across all 24 hours — the data has no diurnal signal to test against, so I couldn't confirm or rule this out statistically. I left `posted_at` as documented (UTC) for Q8 in the absence of contradicting evidence.
- **Cross-portal duplicate properties**: I expected the same physical unit to be re-listed by more than one of the five source portals (`100acres`, `dwelling`, `magichomes`, `squarelane`, `zerobroker`), given they clearly scrape overlapping inventory. I tried three increasingly strict matching signatures (exact coordinates; coordinates+bedroom+floor; a full physical-attribute fingerprint) and found zero genuine duplicates each time. This dataset doesn't have that particular bug — `unique_properties` in my submission is instead computed as `total records - corrupt - fake`, since those are the records that don't correspond to a genuine physical property at all.
- **High-volume repeat phone numbers as a fraud signal**: several contacts appear on 20-30+ listings across 9-10 localities, which looked suspicious at first. Checking their `posted_by` field showed they're consistently labeled `agent` — normal behavior for a real estate agency, not fraud.
- **Cross-role phone number reuse** (same number used as `owner`, `agent`, and `builder` across different listings): logically this shouldn't happen for a real single entity, so I flagged it as a strong fake signal. Testing it against the full dataset showed it flags 4142 of 4700 listings (88%) — clearly just how the dataset's phone number pool is generated, not a fraud signal. Discarded.

## Known uncertainty / what I'd do with two more days

- **`listings_last_7_days`**: computed assuming `posted_at` is genuinely UTC as documented, since I found no way to statistically confirm or refute the timezone claim given the flat hour-of-day distribution. With more time I'd look for an independent way to validate this (e.g. checking whether `is_verified` listings or specific `website` sources have a different, checkable timestamp convention).
- **`projects_with_wrong_listing_count`**: I used "live listings matching project_id" as the comparison basis, since the docs specifically say the field tracks "listings currently available." I'd want to also test whether an actual `/v1/listings?project_id=...` filter call agrees with my client-side computation, in case the filter itself has independent bugs.
- **`fake_listing_ids`**: my final signal (identical contact+locality+bedroom+price on different listing_ids) is narrow and may under-count. I'd explore additional signals with more time — e.g. unrealistic price-per-sqft outliers, or contact numbers whose listings cluster suspiciously by posting time.
- I'd also add automated regression tests against the live API for each documented discrepancy, so the findings stay verified if Ivy Homes' dataset changes between now and review.

## Tools used

Built with the help of Claude (Anthropic) for the data-audit methodology, analysis scripts, and this frontend. All API probing, hypothesis testing, and the final answers/findings were verified against live responses from the actual API for my assigned key.
