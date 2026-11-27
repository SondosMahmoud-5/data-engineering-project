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

1) symbols.csv:
-I noticed here that the total entries are 31 while the unique values for the security ID are only 30 , so there must be a duplicate value.
-The security ID that was duplicated is SEC009, So let's check if it has the same values in all columns or it is a different entry ?
First we need to make sure that all values have the same Format.
-After checking, The only difference between the two duplicate rows is the _source_row column, which is expected since it is a unique identifier for each row. All other columns have identical values, indicating that these rows are indeed duplicates in terms of their data content.     
-We have also noticed that there is only one entry in the "delisting_date" variable, but we decided to keep it for further investigation with other Datasets.
-The dataset contains 30 securities with complete data for the numeric variables shown. The shares_outstanding variable ranges from 41 million to 891 million shares, with a mean of 425.7 million and a median of 411 million, indicating a relatively balanced distribution. The wide range and high standard deviation suggest substantial variation in company size.
-The distribution of shares_outstanding shows substantial variation across tickers. **TAVV** has the largest number of outstanding shares (891 million), while the smallest number is for **ROSS** ticker with shares (41 million). This wide range suggests that the dataset contains companies of varying sizes, which may influence subsequent analyses involving price, trading volume, or market value.
-The shares_outstanding variable shows substantial variation across securities, ranging from 41 million to 891 million shares. The distribution is relatively balanced around a median of 411 million shares, and no significant outliers are observed based on the boxplot analysis.
-The dataset contains 30 unique securities, each represented by a unique ticker symbol.

2) symbol.history.csv:
-symbol history dataset includes only 3 entries for 3 different companies in which there is a change that happened.
-We matched its format with symbols data set.

3) Bars:

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
-For the Open,High,low and close Prices we can observe the following: Prices vary greatly across securities the range between the min value and max value is very large, Mean value is greater than the Median, indicating a right-skewed distribution, as there are some High values dragging the distribution to the right.
-The dataset contains 32 unique securities observed across 3 independent simulation runs (RUN-2025A, RUN-2025B, and RUN-2025C) covering the period from January to June 2025.
-We can also observe that, The median of RUN-2025C is higher than the other two runs, with a wider spread contaning more high price outliers.
while RUN-2025A and RUN-2025B look quite similar.
-Based on the above table:
All 32 tickers are present across the 3 simulation runs. However, the number of observations per ticker varies substantially. While most tickers contain approximately 120-123 daily observations per run, some securities such as ORBT and DOVV contain considerably fewer records, suggesting shorter trading histories or missing data periods.
-The dataset exhibits varying ticker lifetimes. While several securities are present throughout the entire observation window (123 trading days), others appear only during part of the period. For example, DOVV is present only from 12-May-2025 to 30-Jun-2025 (34 trading days), whereas ORBT is present only from 02-Jan-2025 to 14-Mar-2025 (50 trading days). This suggests that the dataset includes securities entering and leaving the trading universe over time.
-The analysis reveals considerable variation across securities in both price and trading activity. INDEX is characterized by the highest average closing price and trading volume, whereas the remaining securities exhibit comparatively lower and more clustered values.
-

