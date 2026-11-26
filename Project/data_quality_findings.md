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