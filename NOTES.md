# What I checked, and what the agent got wrong

## What the agent got wrong

The agent made one silent mistake that I caught by running the acceptance script rather
than trusting its summary: after fixing the `wear_percent` formula it described the
missing-reading default as "treat the car as freshly serviced", but the first version of
that fix still used `car.get("last_service_km", 0)` in `car_wear` inside `fleet_report.py`,
while only fixing the default in `needs_service` in `km_wachter.py`. That meant the report
would still raise a `KeyError` on a car with no reading, even though `needs_service` handled
it correctly. I caught this because `verify.py` reported FAIL on "The nightly report no longer
crashes" even after the `km_wachter.py` fix was applied. I had to direct the agent to also
update `fleet_report.car_wear` to use the same `.get("last_service_km", car["odometer"])`
pattern before that check turned green.

The other thing worth noting: the agent correctly identified `MILES_PER_KM = 1.609` as wrong
but initially described it as "the constant is inverted" without explaining *why* 1.609 is the
wrong direction. The correct value is 0.621371 (miles per km). I confirmed this with a quick
mental check: 100 km is roughly 62 miles, not 161 miles. The agent fixed it correctly once
directed, but the first explanation was vague.

## What I checked before I accepted its work

I ran `python verify.py` after every round of fixes rather than trusting the agent's "done"
message. Specifically:

- I confirmed `SERVICE_INTERVAL_KM == 15000` and `WARN_AT_PERCENT == 80` were untouched
  by reading the constants in `km_wachter.py` directly and checking the verify.py output
  line "The 15000 km / 80% rules are untouched — PASS".
- I verified the wear fix was real by mentally tracing: `14900 / 15000 * 100 = 99.3 %`,
  which is above 80 %, so `needs_service` returns True. Before the fix, `14900 // 15000 = 0`,
  so it returned False. The test output confirmed this.
- For the missing-reading fix I checked both call sites: `needs_service` in `km_wachter.py`
  and `car_wear` in `fleet_report.py`. Both now use `.get("last_service_km", car["odometer"])`.
- I ran `python3 analyze.py` and read the output myself to confirm the correlation table
  matched the claim in the file header before accepting the analysis as done.

## What the data actually said

The obvious assumption — that high-mileage or older cars break down more — is wrong for this
fleet. Both `odometer_km` and `age_years` correlate at r ≈ 0.00 with breakdown. The breakdown
rate is identical (22 %) in the top and bottom halves of both columns.

What actually predicts breakdown is `km_since_service` (r = +0.40). Cars in the top half of
that column broke down at 35 %, versus only 8 % in the bottom half — a 4× difference.
`avg_daily_km` (r = +0.25) and `load_factor` (r = +0.22) add further signal: heavily used,
heavily loaded cars that are also deep into their service window are the highest-risk group.

The risk score weights those three factors by their correlations and captures 13 of 26 breakdown
cars in the top-20 risk slots — 50 % recall using only 17 % of the fleet. That is meaningfully
better than the 80 % wear rule, which would not flag those cars until they were nearly worn out.
