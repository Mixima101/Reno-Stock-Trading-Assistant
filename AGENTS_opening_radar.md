# AGENTS.md

This repository is for building a desktop trading-research assistant for Canadian-listed issuers. The application should help the user browse TSX and TSXV companies, filter them by issuer metadata, and analyze CEO.ca channel activity only in ways that respect CEO.ca’s Terms of Use and any applicable data rights.

The project must be built in small working stages. Always prefer a correct, simple, compliant version over a broad unfinished system.

## Core product goal

Build a sleek desktop application that lets the user view, search, filter, sort, and export a universe of TSX and TSXV issuers.

The table should include normal ticker, NATO phonetic ticker, company name, exchange, sector, sub-sector, listing type, market cap, shares outstanding, current or recent trading volume when available through the configured market-data provider, and later CEO.ca activity metrics when those metrics are available through the configured Parse.bot CEO.ca API.

The application is for personal research support. It must not imply that it predicts returns, guarantees trades, or provides investment advice.

## Primary opening-radar workflow

The first live product workflow should be a controlled list-pulling tool for morning research. The user should choose exactly one scan mode at a time: Top CEO.ca activity or Top trading volume. In the desktop UI, these should be mutually exclusive radio buttons. Selecting one mode must deselect the other.

The user must enter the number of tickers to pull before running the scan. In the UI, use a numeric field named Number of tickers to pull. In the CLI, use a flag such as --list-size. The run should start only when the user presses a button named Gather list or uses the matching CLI command.

The list size must control the number of final tickers shown and the number of tickers that receive Parse.bot discussion-thread calls. The program must not fetch comments for more tickers than the user requested.

In testing mode, default the list size to 2. Testing mode should allow 1, 2, or 3 tickers without additional override. Do not run larger live Parse.bot comment pulls during testing unless the user deliberately changes the testing cap and confirms the estimated credit use.

For a Top CEO.ca activity scan, the app should call Parse.bot get_trending_stocks once, filter the returned symbols against the local TSX and TSXV issuer universe and configured small-cap rules, select the top N tickers from that filtered result, then call Parse.bot get_discussion_thread once for each selected ticker by default. A Top CEO.ca activity scan with N tickers should therefore use approximately 1 + N Parse.bot credits when one discussion page per ticker is used.

For a Top trading volume scan, the app should use the yfinance volume provider to rank the configured TSX and TSXV small-cap universe by recent or opening volume, select the top N tickers, then call Parse.bot get_discussion_thread once for each selected ticker by default. A Top trading volume scan with N tickers should therefore use approximately N Parse.bot credits when one discussion page per ticker is used. yfinance calls do not consume Parse.bot credits.

For both scan modes, the list view should include ticker, company name, NATO phonetic ticker, exchange, sector, market cap when available, recent volume, latest volume bar time, comments in the last 60 minutes, source status fields, and a rank field. The list should support sorting by ticker, company name, recent volume, comments in the last 60 minutes, and rank. Basic filtering should remain available where it is already implemented.

The list view should include a simple visual volume bar for each row. The bar should compare the row's volume to the other rows in the current list. It should be a display aid only and should not be treated as a trading signal. If one stock dominates the list, the UI may use a capped or logarithmic scaling option, but the raw volume number must remain visible.

After gathering a list, clicking a ticker should open a detail screen for that stock. The detail screen should reuse the comments, activity metrics, issuer metadata, and volume data already downloaded during the current run. Opening the detail screen must not trigger another Parse.bot call unless the user presses an explicit Refresh comments button.

The detail screen should show issuer metadata, NATO ticker, CEO.ca channel, scan mode, scan rank, recent volume fields, latest price fields, comments in the last 60 minutes, comment-fetch status, and the actual downloaded comments from the last 60 minutes. For each displayed comment, show timestamp, poster identifier when available, and comment text. Comment text should be kept in memory for the current session by default and should not be written to persistent output files unless the user explicitly enables raw comment retention.

The comment-count calculation should use returned comment timestamps and count comments whose timestamp falls within the configured lookback window, defaulting to 60 minutes. Use timezone-aware datetimes. If the first discussion-thread page does not reach back far enough to cover the full 60-minute window, mark the comment count as partial rather than pretending it is complete. Do not paginate beyond one page per ticker by default.

Before any live Parse.bot run, the app must show the estimated credit cost. For Top CEO.ca activity, estimate one trending call plus one discussion-thread call per selected ticker per page. For Top trading volume, estimate one discussion-thread call per selected ticker per page. The user should see the estimate before confirming the run.

A possible future CLI command for the Top CEO.ca activity route is:

python -m reno_stock_trading_assistant.cli scan-radar --mode comments --live --list-size 3 --post-limit 25 --max-pages 1

A possible future CLI command for the Top trading volume route is:

python -m reno_stock_trading_assistant.cli scan-radar --mode volume --live --list-size 3 --interval 5m --lookback-minutes 15 --post-limit 25 --max-pages 1


## Compliance-first CEO.ca policy

CEO.ca activity access for this project must be handled only through the user’s subscribed Parse.bot CEO.ca API. Do not build any direct CEO.ca scraping, crawling, browser automation, direct endpoint discovery, or direct CEO.ca request logic.

Do not run automated Parse.bot discussion-thread requests against the full TSX or TSXV universe.

Do not create background monitoring loops or repeated unattended runs for CEO.ca activity.

Every live Parse.bot run must have an explicit ticker/channel limit. The app must never silently choose an unlimited or full-universe run.

Do not bypass authentication, paywalls, rate limits, technical controls, access controls, or intended website behavior.

Do not store full CEO.ca comment text by default.

Do not reproduce, republish, redistribute, or compile CEO.ca comments, posts, articles, or other site content into a separate content database.

Do not create functionality that posts, votes, boosts, flags, reports, deletes, or otherwise interacts with CEO.ca user accounts or community features.

Parser development and tests may use local fixture response files already committed to the repository. Treat these files only as fixtures for offline parser tests. Production CEO.ca activity pulls must use the configured Parse.bot API provider.

Live Parse.bot access must remain disabled by default and must require an explicit --live flag or equivalent UI confirmation.

If Codex is asked to add live CEO.ca access, it must implement only the Parse.bot provider path described below.

## Parse.bot-only CEO.ca development

The project may calculate CEO.ca activity metrics from Parse.bot CEO.ca API responses.

The project may define data models for CEO.ca-style activity records.

The project may build a UI that displays activity metrics supplied by the Parse.bot provider.

The project may parse local fixture files for tests, but the production activity provider should be Parse.bot-only.

Do not add alternative live CEO.ca providers.

## Optional Parse.bot CEO.ca API access

Parse.bot may be used as the live CEO.ca activity provider only when the user has an active Parse subscription or credits, the API key is supplied locally, and the run has an explicit user-controlled ticker limit.

Live Parse.bot access must remain disabled by default. The app should run fully from issuer files and test fixtures without any Parse.bot configuration.

Do not hard-code API keys, scraper IDs, endpoint URLs, request headers, local paths, or user account details. Configure them through environment variables, command-line flags, UI settings, or an uncommitted local config file. Use these variable names unless there is a strong reason to change them:

PARSE_API_KEY

PARSE_CEOCA_SCRAPER_ID

PARSE_CEOCA_MAX_TICKERS_PER_RUN

PARSE_CEOCA_TEST_TICKER_LIMIT

PARSE_CEOCA_POST_LIMIT

PARSE_CEOCA_MAX_PAGES_PER_TICKER

PARSE_CEOCA_RATE_LIMIT_PER_MINUTE

PARSE_CEOCA_DRY_RUN_BY_DEFAULT

PARSE_CEOCA_MAX_CREDITS_PER_RUN

RADAR_DEFAULT_SCAN_MODE

RADAR_DEFAULT_LIST_SIZE

RADAR_TEST_MAX_LIST_SIZE

RADAR_COMMENT_LOOKBACK_MINUTES

RADAR_COMMENT_TEXT_RETENTION

The Parse.bot CEO.ca API uses the Parse account API key in the X-API-Key request header. The scraper ID selects the user’s subscribed CEO.ca API copy. Endpoint names select the action to run.

Use this URL shape for HTTP access to the CEO.ca API copy:

GET https://api.parse.bot/scraper/{scraper_id}/get_trending_stocks

GET https://api.parse.bot/scraper/{scraper_id}/get_discussion_thread?channel={channel}&limit={limit}

GET https://api.parse.bot/scraper/{scraper_id}/get_discussion_thread?channel={channel}&limit={limit}&until={timestamp_ms}

GET https://api.parse.bot/scraper/{scraper_id}/get_popular_posts?limit={limit}

GET https://api.parse.bot/scraper/{scraper_id}/get_popular_posts?limit={limit}&until={timestamp_ms}

For this project, prefer get_trending_stocks as the first live Parse.bot call in a run. Then filter the returned symbols against the local TSX and TSXV issuer universe, small-cap criteria, listing type, exchange, watchlist, and any other configured screen. Only after filtering should the app call get_discussion_thread for selected channels.

The number of tickers/channels pulled from Parse.bot must be totally user-controllable. Expose it through a CLI flag such as --max-tickers and later through a UI field named Maximum tickers to pull. This value should control the number of selected tickers that receive get_discussion_thread calls.

Testing defaults must be extremely conservative. In test mode, default to two tickers for the opening-radar workflow. Allow one, two, or three tickers without additional override. Do not test with more than three live Parse.bot discussion-thread calls unless the user intentionally changes the value, sees the estimated credit cost, and confirms the run.

A dry run must estimate the number of API calls before sending any live requests. The estimate should count one call for get_trending_stocks plus one call per selected discussion channel per page. If pagination is enabled, include the additional pages in the estimate before sending requests.

Before a live run, print or display the selected ticker count, the selected tickers if feasible, the configured post limit, the configured page limit, the estimated call count, and the current mode. The run should require confirmation or an explicit --live flag.

Do not call get_discussion_thread for the full TSX or TSXV universe. Do not page deeply through historical comments by default. Do not build unattended monitoring loops against CEO.ca content.

Default live limits should be conservative:

Default list size in test mode: 2

Maximum list size in test mode without additional override: 3

Maximum tickers per run in deliberate live mode: 50 unless the user explicitly configures another value and confirms the estimated credit use

Maximum post pages per ticker: 1

Maximum posts per ticker request: 25 unless the user explicitly configures a higher value

Minimum delay between requests: enough to stay under the configured Parse.bot rate limit

The app should track credits conservatively. It should show estimated credits used before the run and actual successful calls after the run. If the user has a free-credit balance, they should be able to set --max-tickers 1 or --max-tickers 2 and know that testing will not accidentally burn through the balance.

Handle HTTP 429 by stopping the live run and marking source_status as parse_rate_limited. Do not retry aggressively. Handle authentication failure by marking source_status as parse_auth_failed. Handle missing configuration by marking source_status as parse_not_configured.

The provider should not persist full spiel text by default. It may read returned post text transiently if needed to compute immediate features and display the current run's detail screen, but normal stored output should keep only minimal activity fields such as spiel_id, timestamp, channel, poster identifier, bot flag, votes, score, and derived metrics. If text analysis is added later, store summaries or numeric features rather than raw comment bodies unless the user explicitly enables text retention and documents why it is needed.

For the opening-radar workflow, comment responses fetched for the selected list tickers should be cached in memory for the current run. The detail screen should use this cached data. Do not make a second comment API call merely because the user clicked a row.

The Parse.bot provider must be implemented behind the activity provider interface. It must be mockable. Unit tests must not call Parse.bot. Integration tests for live Parse.bot calls must be opt-in, skipped by default, and require explicit local environment variables.

The CEO.ca Parse.bot API should not be treated as a market-data source for trading volume unless the response field is documented and verified. Use yfinance or another configured market-data provider for volume, relative volume, price history, and market cap when those fields are needed.

A possible future CLI dry-run command is:

python -m reno_stock_trading_assistant.cli fetch-activity --provider parse --dry-run --max-tickers 2 --post-limit 10

A possible future CLI live test command is:

python -m reno_stock_trading_assistant.cli fetch-activity --provider parse --live --max-tickers 1 --post-limit 10

A larger live command should still require an explicit ticker limit:

python -m reno_stock_trading_assistant.cli fetch-activity --provider parse --live --max-tickers 50 --post-limit 25

Without --live, the command should estimate calls or use local fixtures only.

## Optional yfinance volume provider

yfinance may be used as the first market-volume provider for prototyping early TSX and TSXV volume screens.

The yfinance provider should be separate from the Parse.bot provider. Parse.bot is for CEO.ca social activity. yfinance is for price and volume candles.

Use Yahoo Finance ticker suffixes when needed. TSX tickers usually use .TO. TSXV tickers usually use .V. The app should map local issuer tickers into the Yahoo Finance symbol format and allow overrides for symbols that do not map cleanly.

The yfinance provider should support configurable intraday intervals such as 1m, 5m, and 15m. The default should be 5m for stability unless the user configures another interval.

The yfinance provider should support a user-controlled ticker limit as well. Use these variable names unless there is a strong reason to change them:

YFINANCE_VOLUME_ENABLED

YFINANCE_INTERVAL

YFINANCE_LOOKBACK_MINUTES

YFINANCE_MAX_TICKERS_PER_RUN

YFINANCE_BATCH_SIZE

YFINANCE_REQUEST_SLEEP_SECONDS

For testing, the yfinance provider should default to one or two tickers, matching the Parse.bot testing behavior. The user should be able to set the ticker count explicitly from the CLI or UI.

The first volume implementation should pull recent intraday candles for the selected tickers and compute these fields when data is available:

yahoo_symbol

volume_source

volume_status

latest_volume_bar_time

latest_volume_bar_volume

volume_5m

volume_15m

volume_60m

opening_volume_so_far

last_price

last_price_time
scan_mode
scan_rank
list_size_requested
parse_estimated_credits
parse_actual_calls
comment_fetch_status
comment_window_status
comments_cached_for_detail

The opening-volume scan should be able to run shortly after the market opens. It should rank selected small-cap tickers by opening_volume_so_far or volume_15m so the user can investigate the names with unusual early activity.

For the Top trading volume route in the opening-radar workflow, yfinance should rank the configured issuer universe first, then only the top N selected tickers should receive Parse.bot discussion-thread calls. This keeps Parse.bot credit use proportional to the user-entered list size.

Do not use yfinance to make trading decisions by itself. Treat it as a screening and research input. Handle missing bars, delayed bars, zero-volume bars, halted tickers, failed downloads, and symbols with no Yahoo Finance coverage by marking volume_status clearly rather than crashing.

A possible future CLI command is:

python -m reno_stock_trading_assistant.cli fetch-volume --provider yfinance --interval 5m --lookback-minutes 15 --max-tickers 2

A combined scan can later use Parse.bot trending data, local small-cap issuer filters, and yfinance recent volume:

python -m reno_stock_trading_assistant.cli scan-open --activity-provider parse --volume-provider yfinance --dry-run --max-tickers 2

## CEO.ca sample response structure

The repository contains a sample response file for a CEO.ca channel payload. Use it only as a local fixture for parser development and tests.

The sample response contains fields such as channel, channel_details, spiels, latest_spiel_id, total_spiels, online, quote, stock_info, and articles.

The spiels list is the relevant local fixture for activity parsing. A spiel item may contain fields such as channel, spiel, timestamp, spiel_id, name, votes, bot, score, boost_count, and booster_count.

For metrics, use only minimal fields needed for activity calculations: spiel_id, timestamp, name or poster identifier, channel, and bot flag. Do not persist full spiel text in normal output.

Timestamps appear to be Unix timestamps in milliseconds. Convert them safely to timezone-aware datetimes.

## Activity metrics

When Parse.bot activity data is available, compute these fields:

channel

ticker

company_name

total_spiels

latest_spiel_id

latest_spiel_time

comments_5m

comments_15m

comments_60m

comments_24h

unique_posters_15m

unique_posters_60m

comment_acceleration_score

online_users

activity_source

api_status

last_checked_at

The first acceleration score should be simple and explainable:

comments_15m divided by max(comments_60m / 4, 1)

This compares the latest 15-minute activity against the average 15-minute rate implied by the latest 60-minute activity. Keep the formula easy to change.

## Issuer data

The repository contains a cleaned TMX issuer CSV. Use this as the main issuer universe.

The exact filename may vary, but it should resemble tsx_tsxv_issuers_clean.csv.

If the filename includes a duplicate-download suffix such as “(1)”, normalize it in the repository to tsx_tsxv_issuers_clean.csv.

The issuer table should support at least these fields:

ticker

company name

exchange

sector

sub-sector

HQ location

HQ region

listing type

listing date

market cap

shares outstanding

interlisted fields

OTC flag

Preserve the original source data. If data is transformed, write transformed files separately and document the transformation.

## Search, sort, and filtering workflow

The user should be able to search by ticker, company name, exchange, sector, sub-sector, and listing type.

The user should be able to filter by exchange, sector, sub-sector, HQ region, listing type, and other categorical fields.

The user should be able to sort by ticker, company name, market cap, shares outstanding, sector, sub-sector, and activity metrics.

Numeric fields such as market cap and shares outstanding should support minimum and maximum filtering. Slider controls are preferred once the UI supports them.

The user should be able to export the currently filtered table to CSV.

The opening-radar UI should include mutually exclusive Top CEO.ca activity and Top trading volume radio buttons, a Number of tickers to pull field, an estimated Parse credits display, and a Gather list button. The result table should support row click-through into the stock detail screen. The UI should make it clear when data came from Parse.bot, yfinance, local issuer metadata, or cached current-run data.

## NATO phonetic ticker field

Add a NATO phonetic version of every ticker.

Use the standard NATO alphabet:

A Alpha

B Bravo

C Charlie

D Delta

E Echo

F Foxtrot

G Golf

H Hotel

I India

J Juliett

K Kilo

L Lima

M Mike

N November

O Oscar

P Papa

Q Quebec

R Romeo

S Sierra

T Tango

U Uniform

V Victor

W Whiskey

X X-ray

Y Yankee

Z Zulu

For display, use lowercase words joined by hyphens.

Examples:

TALA becomes tango-alpha-lima-alpha

DT becomes delta-tango

AEM becomes alpha-echo-mike

If a ticker contains punctuation, convert it into readable words. For example, ABC.A may become alpha-bravo-charlie-dot-alpha.

If a ticker contains numbers, preserve them as digits unless a specific spoken-number mapping is added later.

## Ticker and channel mapping

Create a mapping function that can produce a likely CEO.ca-style channel slug from a ticker, but use that mapping for live activity calls only through the configured Parse.bot provider and only within the explicit ticker limit.

The first mapping rule should lowercase the root ticker and remove common exchange suffixes.

Examples:

TALA.V becomes tala

TALA becomes tala

ABC.A requires careful handling and should be tested.

The app should support a future channel_overrides.csv file that maps issuer tickers to known channel names.

## Recommended architecture

Use Python for the first version.

Recommended libraries:

pandas for CSV ingestion and table manipulation.

pydantic or dataclasses for structured records.

PySide6 for the desktop UI.

pytest for tests.

Do not introduce a database in the first version unless needed. Start with in-memory data and CSV output.

If persistent activity history becomes necessary, use SQLite.

For packaging, target Windows first. Use PyInstaller for an executable. Use Inno Setup or NSIS for a click-to-run installer with a progress bar and desktop shortcut, but only after the core data workflow and UI are working.

## Suggested project structure

Use this structure unless there is a strong reason to change it:

data/input/ for source issuer CSV files.

docs/samples/ for sample response files and discovery notes.

src/reno_stock_trading_assistant/ for application code.

src/reno_stock_trading_assistant/data/ for issuer loading and cleaning.

src/reno_stock_trading_assistant/activity/ for activity data models and metrics.

src/reno_stock_trading_assistant/ceoca/ for CEO.ca fixture parsing and the Parse.bot provider interface.

src/reno_stock_trading_assistant/market_data/ for yfinance volume provider code.

src/reno_stock_trading_assistant/ui/ for desktop UI code.

tests/ for unit tests.

scripts/ for command-line utilities.

packaging/ for future installer and build files.

Do not move user-uploaded source data without preserving it or documenting the change.

## First milestone

Build a CLI prototype that works fully from local files.

The CLI should load the issuer CSV, create NATO phonetic tickers, normalize key fields, parse the local CEO.ca sample response file, calculate activity metrics from the sample, and write a CSV output.

The first CLI command can be simple:

python -m reno_stock_trading_assistant.cli build-table --issuers data/input/tsx_tsxv_issuers_clean.csv --output output/issuer_table.csv

A second command can parse the local sample response:

python -m reno_stock_trading_assistant.cli parse-sample --sample docs/samples/ceoca_get_spiels_tala_sample_response.txt --output output/sample_activity.csv

Do not make live CEO.ca requests in the first milestone.

## Second milestone

Build a local desktop UI.

The UI should show a table with issuer metadata and any locally available activity metrics.

The UI should support search, sorting, dropdown filters, numeric min/max filters, and CSV export.

The table should remain responsive with several thousand rows.

The UI should clearly distinguish issuer metadata from activity metrics and should show when activity data is unavailable, stale, cached, partial, or sample-only.

The first live UI workflow should be the opening-radar list screen. It should allow the user to gather a small, controlled list by Top CEO.ca activity or Top trading volume, then click into a detail screen that reuses the already downloaded comments and volume data.

## Third milestone

Add the Parse.bot activity provider interface, the yfinance volume provider, and the opening-radar scan orchestration.

Create an activity interface that accepts CEO.ca-style activity records from the Parse.bot provider.

Create a market-data interface that accepts recent intraday volume candles from yfinance.

Create a scan service that supports two modes: Top CEO.ca activity and Top trading volume. The scan service should select N tickers, fetch comment threads only for those selected tickers, compute comments_60m, attach yfinance volume fields, and return a list model suitable for the UI.

Both live providers must remain disabled unless explicitly configured and called with a user-controlled ticker limit.

Do not implement any direct CEO.ca scraping or any alternative live CEO.ca source.

## Fourth milestone

Add packaging.

Create a Windows executable and then a Windows installer.

The installer should create a desktop icon and show an installation progress bar.

Keep packaging files separate from application logic.

A possible product name is Channel Radar. The name can be changed later.

## Error handling

Handle missing issuer CSV columns gracefully.

Handle missing activity fields gracefully.

Handle activity records with no timestamp or invalid timestamp.

Handle empty activity files.

Handle duplicate activity IDs.

Handle malformed JSON sample files with a clear error message.

When activity or volume data is unavailable, mark api_status, source_status, or volume_status clearly rather than crashing.

## Testing requirements

Add tests for issuer CSV loading.

Add tests for ticker normalization.

Add tests for NATO phonetic ticker conversion.

Add tests for channel slug mapping.

Add tests for parsing the local sample response file.

Add tests for rolling comment-count calculations.

Add tests for export output fields.

Add tests for Parse.bot call budgeting and max-ticker enforcement.

Add tests for yfinance symbol mapping, volume aggregation, and missing-volume handling.

Add tests for opening-radar scan mode selection, list-size enforcement, Parse.bot credit estimation, comments_60m calculation from timestamps, partial comment-window status, and detail-screen cache reuse.

Tests should use local fixture files and mocks. Tests must not make live Parse.bot or yfinance requests by default.

Any live provider class must be mockable.

## Export fields

When exporting the table, include at least:

ticker

nato_ticker

company_name

exchange

sector

sub_sector

listing_type

market_cap

shares_outstanding

ceoca_channel

activity_source

total_spiels

latest_spiel_time

comments_5m

comments_15m

comments_60m

comments_24h

unique_posters_15m

unique_posters_60m

comment_acceleration_score

online_users

last_checked_at

source_status
yahoo_symbol
volume_source
volume_status
latest_volume_bar_time
latest_volume_bar_volume
volume_5m
volume_15m
volume_60m
opening_volume_so_far
last_price
last_price_time

## Development priorities

Priority one is compliance and data safety.

Priority two is a correct working issuer table.

Priority three is a usable filterable UI.

Priority four is local fixture activity parsing and metrics.

Priority five is the opening-radar list workflow with mutually exclusive Top CEO.ca activity and Top trading volume modes.

Priority six is controlled Parse.bot activity access with explicit list-size limits, comments_60m calculation, cached detail comments, and dry-run budgeting.

Priority seven is yfinance volume integration for ranking and profile volume fields.

Priority eight is desktop packaging and visual polish.

Do not start with installer, icon, or visual polish before the data workflow works.

## Coding style

Keep functions small and readable.

Use clear names.

Prefer explicit data models once parsing is understood.

Avoid over-engineering before the first working version exists.

Add comments where the code handles CEO.ca-specific quirks.

Keep secrets and local files out of Git.

Never commit browser cookies, authorization headers, tokens, session IDs, account data, or local machine paths.

## When unsure

Inspect the repository before making assumptions.

Prefer small pull requests.

Explain what was changed and how it was tested.

Do not add direct CEO.ca website access. Use the configured Parse.bot provider only.

If a user asks for a CEO.ca feature outside the Parse.bot API path, do not implement it as live access.