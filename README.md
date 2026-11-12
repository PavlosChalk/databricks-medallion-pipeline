Pavlos Chalkidis - 05/11

E-commerce Medallion Pipeline
A learning project that models how a business ingests, cleans and reports on e-commerce data using the medallion architecture. Two sources (orders as CSV, clickstream events as nested JSON) flow through Bronze, Silver and Gold, run as one scheduled-style job with data quality checks, and feed a two-page dashboard.

Flow: files in a Unity Catalog Volume → Bronze (raw) → Silver (clean) → Gold (business tables) → dashboard

Layers and tables
Layer	Table	One row per	Notes
Bronze	customers, products	source row	Batch load, everything kept as strings, plus _ingested_at and _source_file
Bronze	orders, events	source row	Auto Loader (incremental), duplicates kept, nested JSON kept as structs
Silver	customers, products	customer_id / product_id	Typed, deduplicated
Silver	orders	order_id	MERGE (re-runnable); invalid rows go to orders_quarantine
Silver	events	event_id	MERGE; nested fields flattened; invalid rows go to events_quarantine
Gold	daily_sales	order date	Revenue from completed and shipped orders
Gold	product_performance	product	Units, revenue, return rate (cancelled excluded)
Gold	customer_summary	customer	Orders and lifetime value
Gold	daily_funnel	date and device	Views, carts, purchases, view-to-purchase rate
Gold	product_engagement	product	Event views, carts, purchases
Gold	product_conversion	product	Engagement joined with order performance
Rules applied in Silver
Orders are valid if quantity > 0 and the timestamp parses. Status is trimmed and lower-cased.
Duplicates are removed by keeping the latest copy by _ingested_at.
Events are valid if event_type is page_view, add_to_cart or purchase, and event_ts and customer_id are present.
Bad rows are quarantined, not deleted, so they can be investigated.
Job: medallion_pipeline
Seven serverless notebook tasks in two lanes: bronze → silver → gold and bronze_events → silver_events → gold_events. silver also feeds gold_events, and gold also feeds gold_events (for product_conversion). quality_checks runs last, after gold and gold_events.

Data quality checks (90_data_quality_checks)
Unique order and event IDs, no non-positive quantities, no orphan customers, valid status and event types, no null customers in events, full product coverage in product_conversion, rates between 0 and 1. A failed check fails the job.

Results
Orders: 26,270 Bronze rows, 26,000 distinct orders = 25,758 in Silver + 242 quarantined
Events: 30,000 distinct events across Silver and quarantine
Re-running the job without new files changes no counts (idempotent)
Problems I hit and fixed
Problem	Cause	Lesson
Silver failed: table not found	A leftover DROP TABLE cell in the Bronze notebook	Keep one-off cleanup out of pipeline notebooks
Silver failed: metadata mismatch	Auto Loader added _rescued_data to the schema	Handle schema changes explicitly (overwriteSchema, drop the extra column)
Job tasks broke after moving notebooks	Tasks reference notebooks by path	Re-select notebooks after moving them
Events Bronze failed: empty directory	Auto Loader needs a file to infer a schema	Generate or land data first
Quality checks could start before data was ready	Missing task dependencies	Declare every upstream table a task reads
Gold daily row count did not grow with new batch	New orders fell on existing dates	Think about the grain of an aggregate
Known limitations
The data is synthetic: events and orders are generated independently, and the revenue chart's early-2026 spike comes from overlapping date ranges in the generator.
No live schedule; the job is run manually.
How to run
Run 00_setup_and_generate (schemas, volume, sample files) once.
Run the job medallion_pipeline.
To test incremental loading, drop a new file into raw_landing/orders/ or events/ and run the job again.
