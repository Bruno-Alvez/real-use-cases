# Technical Decisions

## Parallel migration over in-place correction

The most consequential decision in this project was not a technology choice, it was a migration strategy choice.

In-place correction would have meant running ALTER TABLE statements, adding foreign key constraints to tables with existing violations, cleaning orphan records from a live database while services were querying it. The risk was high: any step that failed mid-way would leave the database in a partially migrated state. Rolling back would be as risky as going forward.

The Strangler Fig approach separated the migration from the production risk. A new environment was built correctly from the start. Data was validated and corrected before loading, not during. The legacy environment stayed untouched and continued serving all active services until the new one was fully verified. If anything went wrong, the fallback was a connection string change.

The cost was time: running two environments in parallel for the duration of the migration. The benefit was that the migration had a clean abort path at every step. For a production system serving an active business, that tradeoff was clear.

## Star Schema over normalized third normal form

The analytics workloads that drive the application require aggregations across time, groupings by customer segment, and joins between orders, products, and customers. A normalized schema would require more joins per query and more indexes to maintain acceptable performance.

Star Schema denormalizes deliberately: dimension tables hold descriptive attributes, fact tables hold transactional events with foreign keys to dimensions. Query patterns become simpler, indexes are fewer, and aggregate functions run against fewer intermediate results.

**Trade-off:** Some values like `ticket_medio` are calculated rather than stored raw. Any calculation produces a derived value that will drift from the source of truth if the underlying data changes. For this use case, the calculated fields are analytics outputs, not operational inputs, so the drift is acceptable and explicitly documented.

## NUMERIC(12,2) over FLOAT for monetary values

Float arithmetic is lossy. A value stored as a FLOAT may not round-trip exactly through addition, subtraction, and comparison operations. For financial data, this produces errors that are small but cumulative and, worse, invisible.

`NUMERIC(12,2)` stores exact decimal values. Aggregations are correct. Comparisons are exact. The cost is 16 bytes per field versus 8 bytes for a FLOAT. For a table with 18,000 order records, that is approximately 2KB difference. The storage cost is not a consideration.

## Foreign keys with RESTRICT over CASCADE

Deleting a customer record that has associated orders should fail, loudly, at the database level. CASCADE would silently remove all the associated orders. Leaving orphan records is what the prior system did, with no constraints at all.

RESTRICT means the application must handle the constraint error explicitly. This is correct behavior: if a customer deletion request arrives, the application needs to decide what to do with their orders before the deletion can proceed. That is a business decision, not a database decision. The database should not make it silently.

## RLS with three role levels

The access model has three levels based on query patterns:

`service_role` bypasses RLS entirely and is used only by ETL Lambdas writing data. `authenticated` has read access to dimensional and fact tables and is used by the web application after a user logs in. `anon` has read access to products, stock, and the FAQ table, which is the minimum required by the conversational agent AI agent to answer customer questions without a user session.

This means the credentials that conversational agent uses cannot read customer PII, even if those credentials were compromised. The narrower the permission scope, the smaller the blast radius of a credential leak.

## Lambda over AWS Glue for ETL

Glue is a managed Spark service designed for distributed processing of large datasets. The e-commerce client warehouse has approximately 72,000 records and grows by 100 to 500 records per day. Spark does not help with 72,000 records. What it does add is a 45-60 second cold start, a cost structure based on DPU-hours, and operational complexity that is not justified by the workload.

Lambda runs in under a second after cold start, costs nearly nothing at this scale, and can be tested locally as a Python function. The constraint is the 15-minute execution limit and the 10GB memory ceiling. Neither affects the current workload.

The one exception is `recalc_clusters`, which runs KMeans clustering via scikit-learn. This function uses a container image via ECR because the dependencies exceed the 250MB Zip limit. The execution model is still Lambda; only the packaging format changes.

## UPSERT over INSERT for all ETL writes

Every write operation in every Lambda uses `INSERT ... ON CONFLICT DO UPDATE`. There are no pure INSERTs in the pipeline.

This makes every Lambda idempotent: running the same function twice produces the same final state. It also means reprocessing historical data to fix a bug, running a backfill after an outage, or retrying a failed execution are all safe operations. The pipeline does not need to track what it has already written; the database handles that via primary key conflict detection.

## Cursor-based incremental sync over full refresh

Each sync Lambda maintains a cursor based on `MAX(data_modificacao)` of the target table. On each execution, it fetches from the source API only records modified after that cursor. This keeps execution times short and predictable as the data volume grows.

The exception is `recalc_rfv`, which runs a full SQL refresh daily. RFV scores depend on relative values across the entire customer base, so a partial refresh would produce incorrect rankings. The full refresh takes under 30 seconds because it is a SQL computation, not an API call.

## Credentials in Secrets Manager over environment variables

The prior system stored Bling OAuth2 tokens as plaintext in the database. Lambda environment variables are encrypted at rest, but they are visible in the AWS console to anyone with Lambda read access and are logged by some infrastructure tooling.

Secrets Manager stores credentials encrypted, access is controlled by IAM policy down to the individual secret ARN, and access is logged in CloudTrail. The `config.py` layer module caches secrets in memory with a 5-minute TTL to avoid fetching on every invocation, which would add latency and cost.
