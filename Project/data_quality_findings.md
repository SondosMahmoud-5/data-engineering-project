Data-quality findings: 

bars_1m.parquet File:
1. Missing close prices: 854 rows
Found with isna().sum(). Rejected as MISSING_REQUIRED_FIELD.
2. Negative volume: 853 rows
Volume can't be below zero. Found with volume < 0. Rejected as INVALID_NUMERIC_RANGE.
3. Invalid OHLC: 1,282 rows
The high isn't the highest price or the low isn't the lowest. Rejected as INVALID_OHLC.
4. Duplicate bars: 15,120 rows
The same run_id, ticker and ts appear more than once. Kept the copy with the earliest received_at and rejected the rest as DUPLICATE_BAR.
5. Unknown run: 260 rows
RUN-TEST isn't a real run in sim_runs.json. Rejected as UNKNOWN_RUN.
6. Mixed timestamp formats in ts, 115 not a time , 151 invalid date and 159 placeholders
Different lengths and formats; some length-10 values aren't times at all. Valid formats are converted to local time. Unreadable ones are rejected as MALFORMED_TIMESTAMP. Rejected as UNKNOWN_RUN.


Duplicate runs: sim_runs.json contains two records for RUN-2025B with the same configuration fields but different created_at values. Detected by checking for repeated run_ids. For the bars merge, I used one RUN-2025B record with created_at excluded, since the other run fields matched.

events.json File:
Events outside the company's listing period: 3 rows
A company can't hold an earnings announcement on an exchange before it's listed there, or after it's left. So on those dates, these tickers matched no company valid on the date.
EV0016 (CELK, 14 Feb) and EV0021 (DOVV, 6 Mar) happened before the company was listed. EV0055 (YARL, 30 May) happened after the company was delisted (30 Apr). Found by matching each event's date to its company's listing period. Rejected as UNKNOWN_SYMBOL.


4) exchange_calendar.csv
- Finding 1: Booleans and dates are text
    - data-quality issues:
        - is_trading_day was stored as text instead of Boolean values. 
        - date was stored as text instead of datetime values.
    - Both convert cleanly: 
        - no date failed `to_datetime(errors="coerce")`
        - no boolean failed to map
    - Decision: Convert columns explicitly to their appropriate data types and check for conversion failures
- Finding 2: Coverage is complete
    - dataset contains every expected trading-calendar date within the period (129 unique dates from 2025-01-02 to 2025-07-01), without gaps or unexpected dates.
    -  no weekends as stock exchanges don't trade on weekends
    - no missing weekdays
    - one time zone (`America/New_York`), which makes date and time comparisons more reliable
    - Decision: Use it as delivered as it is complete
- Finding 3 Five holidays (US Market Holidays), and their missing session hours are structural
    - The 5 missing `session_open` / `session_close` values are exactly the 5 closed days (missing rate 100% when closed, 0% when open): 01-20, 02-17, 04-18, 05-26, 06-19.
    - These are not lost data. Any bar dated on one of these days should not exist.
    - Decision: 
        - Keep the nulls & don't fill these missing values. 
        - If a price bar appears on one of these five holidays, it should be classified as: `OUTSIDE_SESSION`
- Finding 4: One trading day has early close: 2025-04-17 ends at 13:00
    - 123 trading days in the batch window. 122 have 390 minutes (09:30–16:00); 04-17 has 210 minutes (09:30–13:00).
    - Bars between 13:00 and 16:00 on 04-17 are outside the session, and the expected-bars count for that day is 210, not 390.
    - Decision: Read session length from the calendar per day and do not assume 390
- Finding 5: Daylight saving time starts on 2025-03-09
    - Opening time changes during march: The 09:30 New York open is 14:30 UTC until 03-07 and 13:30 UTC from 03-10.
    - A source that stores UTC shows a one-hour jump in its opening time in March
    - Decision: Convert time zones with a real time-zone library (`tz_localize` / `tz_convert`), not with a fixed offset


6) sim_runs.json
- Finding 1: RUN-2025B is in the file twice
    - 4 objects for 3 runs. The two RUN-2025B objects differ only in `created_at` (07-02 and 07-03), so they share the key `(run_id, config_version)`.
    - The run configuration is a Type-2 dimension keyed on `(run_id, config_version)`. Two rows with the same key would duplicate every RUN-2025B bar on a join.
    - Decision: Keep the latest `created_at` and log the older copy as a re-export
- Finding 2: Unit error: RUN-2025C has `volatility.sigma_annual = 18`
    - Runs A and B have 0.18; C has 18. That is a percentage written where a decimal is expected
    - Any comparison of configured against measured volatility would be wrong by a factor of 100 for run C. this is a labelling error in the config, not in the data
    - Decision: Flag it. do not rescale (non fatal problems are kept and flagged). Certification measures the output, so it does not change any metric.


7) vendor_daily.csv
- Finding 1: Structurally clean
    - 10,831 rows, no nulls, no duplicate keys, all runs and tickers known, every date a trading day (no 01-20 rows), OHLC valid, no negative values
    - The vendor file's problems are about conventions, not broken rows, so they only show when we compare it with the minute bars
    - Decision: Accept all rows