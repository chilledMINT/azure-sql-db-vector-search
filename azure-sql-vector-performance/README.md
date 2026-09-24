# MSSQL Vector Performance

A small sample for downloading YFCC image vectors, loading them into MSSQL, and
measuring vector index builds, search latency, and recall.

Downloading, native BCP conversion/loading, index-build measurement, and search
latency/recall measurement are implemented. Offline tests do not establish
successful live execution. Build and search still require validation on the
target server before their results can be used.

Each stage has its own Python file. [download.py](download.py) owns downloading,
its command-line options, and its error handling. [benchmark.py](benchmark.py)
only routes commands to stage entry points. [load.py](load.py) owns the load
stage. [build.py](build.py) runs [scripts/build-index.sql](scripts/build-index.sql)
using a dedicated pyodbc connection and saves build results. [search.py](search.py) uses pyodbc and
[scripts/vector-search.sql](scripts/vector-search.sql) for paired search passes.

### Disclaimer
These scripts and accompanying results are provided for **informational and illustrative purposes only**. They are intended to demonstrate how vector index build and vector search performance can be measured for Azure SQL Database under the specific configurations and workloads described in this repository.

Performance results were obtained using the specified dataset, workload, database configuration, compute configuration, and test environment. **Actual performance may vary** depending on factors including data characteristics and size, vector dimensions, index configuration, query patterns, concurrency, service tier, compute resources, system load, and other environmental conditions.

The results presented here should **not be interpreted as a performance guarantee, service-level commitment, or prediction of performance for a particular production workload**. They are also **not intended as comparative benchmark results against other products or services**.

Customers should evaluate performance using their own data, workloads, configurations, and production requirements before making deployment or capacity-planning decisions.

**MICROSOFT MAKES NO WARRANTIES, EXPRESS OR IMPLIED, WITH RESPECT TO THE INFORMATION, SCRIPTS, OR RESULTS PROVIDED HERE.**

## Download

The download command uses Python's standard library. No packages, SQL instance,
database credentials, or external download tools are needed. Python 3.11 or newer
is required. It has been tested with Python 3.14.

From the repository root:

```powershell
python azure-sql-vector-performance/download.py --size 1M
```

The existing command remains available through the dispatcher:

```powershell
python azure-sql-vector-performance/benchmark.py download --size 1M
```

Choose `1M`, `10M`, or `100M`. Size names are case-insensitive. The default
destination is `data/yfcc-images-SIZE` beside the script, regardless of your current
working directory. To choose a different parent directory:

```powershell
python azure-sql-vector-performance/download.py --size 1M --data-dir D:/vector-data
```

Each dataset folder contains:

```text
base.bin
queries.bin
groundtruth.bin
manifest.json
```

The command uses `urlopen` to copy one file at a time in 1 MiB chunks. It prints
progress and average transfer speed. Each size selects a complete published
corpus and a size-specific ground-truth URL, without cropping or header changes.
The published 1M ground truth has a known compatibility issue described below.

`get_yfcc_urls(size)` uses an explicit paired filename map and `urljoin`;
unsupported sizes are rejected before creating output or making requests.
All sizes share `yfcc100m_query_vecs.fbin`, containing 100,000 queries.
Before any transfer, every missing file's URL must return HTTP 200 to a HEAD
request. Failed checks stop without downloading payloads or changing existing
partial files; there is no fallback or retry. Completed files are skipped without
network checks. Each subsequent GET must also return HTTP 200 before its partial
file is opened for writing.

The 1M, 10M, and 100M downloads need about 5.71 GB, 51.79 GB, and 506.12 GB of disk
space respectively, including queries and ground truth. These are decimal units.
The 100M label represents 98,735,605 actual document vectors, not 100,000,000.

## Data Format

The source is `YFCCImagesDataset` from
[Big ANN Benchmarks](https://github.com/harsha-simhadri/big-ann-benchmarks/blob/main/benchmark/datasets.py).
It provides 1,280-dimensional float32 document and query vectors, 100,000 queries,
Euclidean distance, and a separate ground-truth URL for each corpus size.
The public headers checked during development report 100 neighbors per query.
The downloader records the depth read from the selected file instead of imposing
a hard-coded k limit.

Files are served publicly at
`https://comp21storage.z5.web.core.windows.net/yfcc100m_images/`.
The 1M and 10M files are the publisher's sampled corpora, not prefixes of the full
corpus. Existing Wikipedia downloads are left untouched and are not used here.

Every binary file starts with two little-endian uint32 values: row count and
width. Vector files then contain row-major float32 values. Ground truth contains
all row-major int32 neighbor IDs, followed by all row-major float32 scores.

All three files are copied without conversion. The manifest records the source
URLs, shapes, byte counts, and SHA-256
checksums of the final local files. These are local integrity records, not
publisher-supplied checksums or proof of nearest-neighbor correctness.

The copier requires HTTP 200 and checks byte count when `Content-Length` is provided.
Without that header, it reads until the response ends. Binary headers are read
for the manifest. Deep dataset and ground-truth audits belong in local development
tests, not this command. Dataset licensing must be checked before redistributing downloaded data.
Downloads are excluded from Git by this sample's ignore rules.

On 2026-09-24, HEAD checks returned HTTP 200 for all seven distinct published
URLs (three corpora, three ground-truth files, and the shared query file).
This establishes availability only, not ground-truth correctness.

### Known 1M Ground-Truth Issue

On 2026-09-24, the published `yfcc100m_query_gt100_sampled_1m.bin` returned
neighbor ID 8,572,694 at query 0, rank 1. This cannot reference the 1M document
table, whose IDs are 0 through 999,999. The local file matched its recorded
SHA-256, and its first 408 bytes (header and first query's IDs) matched both the
publisher's 1M and 10M URLs. This comparison does not establish that the entire
two remote files are identical or that the 10M ground truth is correct.

The loader intentionally rejects these out-of-range IDs. Do not remap, truncate,
or discard them to force an import: that would invalidate recall. A corrected
publisher file or separately computed exact ground truth for this corpus is
needed before recall testing. Neither is supplied automatically by this sample.

If this error occurs after documents and queries load, their committed rows
remain. Do not restart the whole load or delete those tables. Index-build timing
depends only on the complete document table and can proceed after its population
checks pass; the load as a whole and ground-truth-dependent search remain incomplete.

## Existing Files And Failures

Completed files are skipped. A completed dataset and its existing manifest are
left unchanged, without rechecking their hashes. If files beside an existing
manifest are missing, the command stops. Use a new `--data-dir` for a fresh copy.

Transfers are written to `.part` files and renamed only after success. A failed
transfer exits with a nonzero status. Rerunning restarts that partial file from
the beginning and skips files that already finished. This is not byte-level
resume. There are no automatic retries or parallel transfers.

The manifest is written after all three files finish and their metadata is read.
If a downloaded file is malformed, inspect it and move it aside before rerunning.

## Load

[load.py](load.py) validates source headers and tools, creates empty tables,
then converts, imports, and verifies one temporary native BCP chunk at a time.
Only successfully verified chunks are removed. Its small helpers stay in the
same file; Python uses only the standard library. No probe tables, format
exports, password cache, or automatic retries run as part of loading.

You need an existing user database supporting `VECTOR(1280)`, a SQL login or
Windows identity with table-creation and BCP permissions, and BCP and sqlcmd in the selected
environment. The wrapper does not create a database, enable preview features,
or change server TLS settings. For a local or test server whose certificate is
not trusted by the client, pass `--trust-server-certificate`. This opt-in switch
passes the trust-certificate option to BCP and sqlcmd; it is off by default.

From PowerShell at the repository root, using Windows tools and the current
Windows identity (no password prompts):

```powershell
python azure-sql-vector-performance/load.py --size 1M --server localhost,1433 --database MSSQLVectorBenchmark --trusted-connection --trust-server-certificate
```

Supply `--bcp` and `--sqlcmd` paths if the desired versions are not on PATH.
`--trusted-connection` and `--username` are mutually exclusive. For a server with
a trusted certificate, omit the local/test `--trust-server-certificate` option.

Alternatively, using SQL authentication with tools in WSL Ubuntu:

```powershell
python azure-sql-vector-performance/load.py --size 1M --wsl Ubuntu --server YOUR_SERVER --database YOUR_DATABASE --username YOUR_LOGIN
```

Use a server address reachable from WSL. WSL's `localhost` does not always refer
to the Windows host. Paths to SQL scripts and native files are translated with
`wslpath`. BCP and sqlcmd default to `/opt/mssql-tools18/bin/` in WSL. Override
them with `--bcp` and `--sqlcmd` if your installations are elsewhere.

When running Python directly inside WSL or on Windows with Windows tools, omit
`--wsl`. The tools are then found on that environment's PATH, unless their paths
are supplied explicitly. The dispatcher also supports `benchmark.py load` with
the same arguments.

Use the same `--data-dir` parent directory as the download command. The default
is `data` beside the script. `--chunk-rows` defaults to 100,000 source rows per
temporary file. For vectors these are documents or queries; for ground truth
these are queries, each with all of its neighbors. `--batch-size` independently
defaults to 10,000 imported rows per BCP transaction. One BCP invocation handles
the whole chunk, not each small read buffer.

At 1,280 dimensions, the default vector chunk is at most 513,400,000 bytes
(about 490 MiB). At depth 100, a full ground-truth chunk is 160,000,000 bytes
(about 153 MiB). Python validates and writes one source row at a time, so it does
not hold the whole chunk in memory. Larger chunks mean fewer connections and
password prompts but more temporary disk use. Rejected-row files can require
additional space; database and transaction-log storage are separate.

Before converting payloads, the loader checks all three headers and exact file
sizes: the selected corpus count, 100,000 queries, matching 1,280 dimensions, a
positive ground-truth depth no larger than the corpus, and matching GT query
count. Depth comes from the GT header, not a hard-coded k limit. Truncated or
extra bytes cause failure. Non-finite vector values, out-of-range or duplicate GT
neighbors, and non-finite or negative GT distances are rejected as chunks are
encoded. A malformed later chunk can therefore stop after earlier chunks commit.

For size `1M`, the tables are `dbo.yfcc_1M_documents`, `dbo.yfcc_1M_queries`, and
`dbo.yfcc_1M_groundtruth`. Other sizes use the same naming pattern. Creation SQL
is in [scripts/setup/create-table.sql](scripts/setup/create-table.sql).
The setup script confirms the selected user database and prints the actual
server build, server identity, endpoint, database, and login. It refuses existing
destination objects and checks column order, types, vector dimensions/float32,
nullability, and absence of identity/computed columns before committing creation.
Nothing is dropped or truncated. Use a dedicated destination with no concurrent
writers; there is no resume or merge into existing tables.

After each import the loader checks BCP's exit status and rejects any nonempty
error file, even with exit code zero. Then
[scripts/setup/load-data.sql](scripts/setup/load-data.sql) checks the exact row
count for that chunk's ID range and throws on mismatch. On a table's final chunk,
it also checks the entire table count. SQL success is required before removing
the chunk and its empty error file. `-m 1` alone is not a zero-rejection guarantee.
These SQL scripts use sqlcmd variables and are executed by the wrapper. They
are not ordinary SSMS query-window scripts.

Document and query IDs are their zero-based source row positions. Ground truth
uses zero-based query IDs, one-based ranks, the original neighbor IDs, and the
original float32 distances. A 1M load should contain 1,000,000 documents, 100,000
queries, and 10,000,000 ground-truth rows.

With `--username`, each SQL connection prompts for the password in the terminal. Expect a setup
prompt and two prompts per chunk (BCP import and SQL verification): 25 prompts
for 1M at the defaults. The wrapper does not accept a password argument, cache
passwords, write credentials to files, or inherit `SQLCMDPASSWORD`/`OSQLPASSWORD`.
Run it in an interactive terminal and enter passwords there. On Windows,
`--trusted-connection` instead uses sqlcmd `-E` and BCP `-T` with the current
Windows identity; no SQL password is requested or cached.

The `bcp` folder under the selected dataset stages only the current chunk.
Conversion leaves the downloaded binaries and manifest unchanged. An existing
`bcp` folder is never reused or overwritten, even if empty. On failure, the
current `.bcp` and error file remain; an incomplete conversion keeps `.bcp.part`.
Successful earlier chunks have been removed and their committed rows remain in
SQL. Do not blindly replay a failed chunk: some of its BCP transactions may
already have committed. Inspect the files and database before deciding on a
manual recovery. The loader performs no failure cleanup or automatic replay.

The loader requires BCP to report version 18.6.1.1 or newer, the documented
minimum for native float32 vector transfer with `-z0`.
See the [BCP reference](https://learn.microsoft.com/en-us/sql/tools/bcp/bcp-utility#data-representation).
Using `-n` alone does not establish that vector values are transferred in native
vector binary format. The wrapper checks BCP and sqlcmd help for required
options, validates WSL path translation/readability when selected, and passes
`-n -z0`. Old or unrecognized version output fails closed; there is no JSON
fallback. In particular, the previously observed Windows client reporting
`17.0.4055.5` fails this strict gate even though its help advertises `-z0`.
Select a tool meeting the version gate before attempting a live load.

The converter uses the documented
[two-byte vector length prefix](https://learn.microsoft.com/en-us/sql/relational-databases/import-export/specify-prefix-length-in-data-files-by-using-bcp-sql-server)
and the vector header layout in
[Microsoft's driver source](https://github.com/microsoft/mssql-jdbc/blob/main/src/main/java/com/microsoft/sqlserver/jdbc/VectorUtils.java).
It copies the float32 payload bytes unchanged. Local tests check the record
layout, values, IDs, ranks, and command sequence. The clearer prefix expression
`struct.pack("<HBBI2x", vector_bytes + 8, 0xA9, 1, dimensions)` preserves the
observed bytes. This refactored load path still needs live validation of the SQL
scripts and native imports on the intended build. No vector index is created.
Row-count checks are not value readback: there is no mandatory full-corpus export.
Any sampled value validation must be run separately and report its sample size.

## Build

One invocation measures one fresh `CREATE VECTOR INDEX` on the already-loaded
document table. Downloading, loading, table/primary-key creation, and search are
separate stages. The build runner uses one dedicated pyodbc connection with
autocommit enabled and Microsoft ODBC Driver 18 for SQL Server. BCP remains the
bulk-load tool; no data is inserted by the build stage.

Install the Python dependency from the repository root:

```powershell
python -m pip install -r azure-sql-vector-performance/requirements.txt
```

Install Microsoft ODBC Driver 18 separately in the environment where Python
runs. Windows Python needs the Windows driver; Python inside WSL needs Linux
pyodbc, unixODBC, and the Linux Driver 18 installation in that distribution.
Windows SQL clients do not provide the driver to WSL. The build runner does not
launch WSL or sqlcmd. Windows authentication works in PowerShell without password
prompts; Linux integrated authentication requires its own Kerberos setup.

From the repository root, after the document load has finished:

```powershell
python azure-sql-vector-performance/build.py --size 1M --server tcp:localhost,1433 --database MSSQLVectorBenchmark --trusted-connection --trust-server-certificate --maxdop 1 --output-dir azure-sql-vector-performance/data/build-runs/1M-first
```

The dispatcher also supports `benchmark.py build` with these arguments. The
former build-only `--sqlcmd` option is removed. Use `--username` instead of
`--trusted-connection` for one hidden password prompt when opening the connection.
Passwords are not accepted on the command line, written to files, or cached.
The connection string is never logged. Driver pooling and reconnect retries are
disabled; all work and result-set consumption use this one connection.

Encryption and certificate validation are enabled by default. The example
explicitly trusts the local test server certificate; omit
`--trust-server-certificate` for a correctly trusted server. Actual engine version
is always recorded. Only supply `--expected-engine-version 18.0.251.0` when an
exact version pin is intended; version equality is not required by default.

`--maxdop` is required: the sample has no implicit parallelism policy. The
example explicitly chooses 1; 0 allows server-selected parallelism and does not
mean one worker. The metric is always Euclidean to match YFCC, the index type
is explicitly DiskANN, and the default name is `yfcc_vector_index`. Other build
options are omitted and recorded as server defaults without guessing their
effective values. `--repetition` defaults to 1 and is a result label, not a loop.

The exact CREATE statement is printed and retained. For the example it is:

```sql
CREATE VECTOR INDEX [yfcc_vector_index] ON [dbo].[yfcc_1M_documents] ([embedding])
WITH (METRIC = 'euclidean', TYPE = 'DISKANN', MAXDOP = 1);
```

### Timing Scope

The metric is **server-side CREATE VECTOR INDEX elapsed time**. The SQL script
places the timer immediately around the dynamic CREATE statement:

```sql
SET @started = SYSUTCDATETIME();
EXEC sys.sp_executesql @create_sql;
SET @finished = SYSUTCDATETIME();
SELECT CONVERT(FLOAT, DATEDIFF_BIG(NANOSECOND, @started, @finished)) / 1000000000.0 AS build_seconds;
```

This includes server-side dynamic statement dispatch/compilation, execution,
and waits until CREATE returns. It excludes connection/login, Python/driver
overhead and client transport, validation, any approved DROP, post-build metadata,
and output export. It is neither server CPU time nor the Python
`perf_counter()` client-observed measurement. It does not use the search
benchmark's statement-completed XEvents.

`SYSUTCDATETIME()` is a wall clock, not monotonic; nanosecond arithmetic does not
imply nanosecond clock accuracy. Clock adjustments can affect the duration; a
negative duration is rejected. Timer placement is checked offline, but execution
and clock behavior on the target build still require live validation.

### Checks And Results

The workflow is connect, validate, collect metadata, execute CREATE, verify, and
export. The runner refuses an existing output directory before connecting. Small
parameterized queries validate the target database, SELECT/ALTER permissions,
`vector_dimensions` and `vector_base_type` for non-null VECTOR(1280) float32
storage, and the existing clustered INT primary key. User databases are not
classified by numeric database ID. Expected document/non-null counts and source
ID range are checked before CREATE and afterward. Incomplete loads stop before
index changes. Counts use the actual population, including 98,735,605 documents
for size 100M.

SQL values are bound through `?` parameters. SQL identifiers cannot be bound:
the runner restricts table/index identifiers to letters, digits, and underscores
and brackets them before building the visible CREATE statement. That complete
statement is passed as a bound NVARCHAR value to the short timer script. All
result sets are read and `cursor.nextset()` is called through completion so late
SQL errors are not missed. The result dictionary is built in Python, with no
SQLCMD substitutions, phase branching, or prefixed JSON message parsing.

Existing indexes are refused by default. `--replace-existing` enables a prompt
requiring the exact index name before removing only that vector index. A
differently named index is never dropped; a non-vector index with the requested
name is refused. On the same connection, the runner rechecks the table/index
identity after confirmation. Any approved DROP is a separate statement outside
the timed SQL batch.
No loading or concurrent writes should occur during a build. Validation reads can
warm pages; there is no warm-up build, cache clearing, or cold-cache claim.

After CREATE, the runner verifies the vector index, table, column, type, metric,
and unchanged population outside the timer. Each new attempt directory retains:

- `request.json`: run ID, repetition, dataset, expected population, dimensions,
	storage type, metric, index name, MAXDOP, omitted-option policy, exact SQL, and
	timing label.
- `create-result.json`: the server's CREATE completion flag, start UTC and
	elapsed time, actual population, engine version, and collected configuration,
	saved before postchecks. Completion does not imply postcheck/export success.
- `result.csv`: a header and one result row, including actual row count, start
	UTC, elapsed seconds, settings, and success/failure status. Nested settings
	and metadata are JSON text within quoted CSV cells; missing values are blank,
	not zero. Commas, quotes, and line breaks are escaped by Python's CSV writer.
- `result.json`: the combined result with actual row count, start UTC, elapsed
	seconds when CREATE completed, and success/failure status, retained for recovery.

CSV export is outside the timed interval and uses exclusive file creation.
Failed attempts also get a CSV when writing is possible. Existing or partial
exports are never overwritten, and an export error never causes another CREATE.

No SQL result table is created. Each attempt directory is reserved before work
starts, so failed attempts also occupy that destination. Repetitions require a
new destination and a fresh CREATE, including another replacement confirmation
when necessary. No operation is automatically retried.

A failed or interrupted CREATE is not a successful duration. If CREATE returned
but postchecks/export failed, its completion checkpoint and elapsed time are
retained without claiming a verified successful result. Original exceptions are
re-raised by the runner, even if failure export or cleanup also fails; saved error
messages redact the supplied password. The CLI prints only the exception type
and directs you to the retained result. Inspect database state before retrying.
Recover retained reporting without connecting or rebuilding:

```powershell
python azure-sql-vector-performance/build.py --recover-result azure-sql-vector-performance/data/build-runs/1M-first
```

Recovery reads the request, CREATE checkpoint, and final result when available,
then writes new `recovered-UUID.json` and `recovered-UUID.csv` files without
overwriting an earlier file. A
partial export is skipped in favor of the last intact checkpoint. This command
does not parse old sqlcmd logs. An incomplete checkpoint remains incomplete; it
cannot prove completion if the connection was lost before the result arrived.
Index sizes, CPU, and progress metrics are not collected and
are left absent rather than reported as zero. Service configuration records the
full engine version/edition, database compatibility, and database MAXDOP and
preview configuration where available, not an inferred effective worker count.

## Search

Search uses already-loaded YFCC tables and an existing compatible Euclidean
DiskANN vector index. It never downloads, loads, creates/replaces an index, or
replays a failed query. Install its only Python dependency and Microsoft ODBC
Driver 18 for SQL Server:

```powershell
python -m pip install -r azure-sql-vector-performance/requirements.txt
```

After completing load and build, first obtain approval for a small live run.
This example selects 10 queries and one measured repetition for validation;
it still includes the single discarded warm-up pass:

```powershell
python azure-sql-vector-performance/search.py --size 1M --server tcp:localhost,1433 --database MSSQLVectorBenchmark --trusted-connection --trust-server-certificate --expected-engine-version 18.0.251.0 --dimension 1280 --metric euclidean --query-mode ann --k 10 --query-count 10 --repetitions 1 --maxdop 1 --output-dir azure-sql-vector-performance/data/search-runs/1M-small-first
```

The dispatcher supports `benchmark.py search` with the same options. The current
query mode is `ann`, using `SELECT TOP (@k) WITH APPROXIMATE` and
`FORCE_ANN_ONLY`; there is no silent legacy TOP_N or exact-search fallback. This
syntax and the event schema must be verified against the intended local build.
Other syntax versions require an explicit design change, not automatic replay.

For the full protocol, defaults are `k=10`, `query_count=1000`, one discarded
warm-up pass, ten measured repetitions, and `MAXDOP=1`. Unlike the build stage,
search fixes MAXDOP to 1. Query count and k are bounded by the dataset manifest
and validated database coverage; repetitions can be reduced for an explicitly
declared small test. The settings and full server version are retained.

Windows authentication uses the current identity without prompts. SQL
authentication uses `--username` and one hidden password prompt to open the
dedicated autocommit connection; there is no credential cache. Connection
pooling and driver reconnect retries are disabled. The same workload connection
performs validation, stamping, and every search, and all result sets are consumed
so late errors are not missed. Use a dedicated database without concurrent writers.

### Workload

[scripts/validate-search.sql](scripts/validate-search.sql) checks the document
population, ID coverage, non-null vectors, matching float32 dimensions, and the
existing index/metric. Selected query IDs must cover 0 through query_count-1
exactly once. Every selected query needs GT ranks 1..k, unique neighbor IDs, and
neighbors within the selected corpus. The retained download manifest declares
the corpus and Euclidean metric; these checks are not independent proof of the
publisher's ground-truth correctness. Validation reads can warm pages.

Each pass submits one server-side loop batch. For each query, sequentially:

1. Read the embedding into `@q` and clear `@found`.
2. Stamp LATENCY, execute the timing SELECT into `@sink`, and immediately save
	`@@ROWCOUNT`; no neighbor result set goes to the client.
3. Stamp RECALL and execute the same logical search into `@found`.
4. Fail if the two row counts differ, then count overlap with GT top-k.
5. Record query ID, returned/matched/GT counts, and NULL latency for later XEvent
	attribution before advancing to the next query.

The stamp is 16 opaque run-ID bytes, four big-endian query-ID bytes matching
SQL `CAST(query_id AS BINARY(4))`, and one kind byte (`01` LATENCY, `02` RECALL).
Every measured repetition has a distinct run ID. UUID bytes are supplied as a
binary literal, avoiding SQL uniqueidentifier byte-order conversions.

### Timing Evidence

Per-query latency is the timing SELECT's public `sql_statement_completed` or
`sp_statement_completed` XEvent duration, stored in microseconds. No Python or
SQL wall-clock delta substitutes for this metric. This differs from the build
stage's server-side wall-clock timer. Target metadata must confirm both event
schemas, statement fields, CONTEXT_INFO/session actions, and microsecond units.

Each measured repetition starts its own session before the loop. The filter uses
the workload session ID and the event's own `statement` field containing the
timing-only run-marker comment, never whole-batch `sql_text`. The parser decodes
the stamp rather than using event order, plan hashes, or query position. It
requires exactly one timing event for every selected query and rejects duplicates,
missing/invalid durations, unexpected attribution, dropped events or buffers,
pending buffers, overwritten events, and truncated XML.

The ring buffer and drop counters are read and saved while the session is still
active, before STOP discards its memory. Reading the target DMV requests a flush;
any incomplete snapshot fails validation rather than guessing or replaying work.
The runner stops and drops only its own newly-created collection session after
evidence is saved, including on workload failure. If evidence cannot be saved,
the session is retained and its name is recorded for manual inspection/cleanup.

SQL Server/Managed Instance use server-scoped sessions; Azure SQL Database uses
database-scoped sessions. The runner checks `ALTER ANY EVENT SESSION` and
`VIEW SERVER PERFORMANCE STATE`, or `ALTER ANY DATABASE EVENT SESSION` and
`VIEW DATABASE STATE`, respectively. Table SELECT permissions are also required.
No permissions, internal trace flags, or preview options are enabled automatically.

The 4 MiB ring target does not eliminate the approximately 4 MiB XML serialization
limit. Large query counts or statement payloads can overflow it. Such attempts
fail with retained evidence; the sample does not silently sample, change targets,
split one repetition into client batches, or retry. Validate a small run before
scaling. No internal fallback events are required, and `FORCE_ANN_ONLY` alone is
not reported as separately proven absence of fallback.

### Recall And Summaries

Recall@k is distinct captured IDs overlapping GT top-k divided by its validated
count k. Three correct results for k=10 means 30%, not 100%. Successful short or
empty answers remain in summaries. Failures and missing timing evidence never
become empty answers. Each repetition reports arithmetic mean per-query recall,
returned<k count/percentage, and p50/p95 from individual latencies. Percentiles
use linear interpolation at `(n - 1) * p` (type 7); milliseconds are microseconds
divided by 1000.0. No combined percentile of repetition averages is reported.

Timing physical reads and, when present, page-server reads are retained. Hot
means zero physical reads and, on Hyperscale, zero page-server reads. Missing
required counters mean unknown, not zero. Unknown Azure service edition is
conservatively treated as requiring page-server reads. Read-bearing results are
kept and counted; they are never discarded or rerun until fast. A warm-up pass
alone does not establish hot execution.

Latency and recall come from separate executions. Equal row counts do not prove
identical neighbors, and the recall execution can benefit from cache effects of
the timing execution. This paired workload is not a single-search throughput test.

Each new output directory contains settings, target metadata, the source manifest,
preflight observations, and per-pass workload SQL. Each measured pass retains
raw result sets, ring-buffer XML, event counters, attributed per-query results,
and its summary/status. A failed pass retains available partial results and stops
later passes without overwriting or deleting prior attempts. Warm-up rows are
retained for diagnostics but excluded from measured summaries.

Each successfully validated measured repetition also writes `queries.csv`
beside `queries.json`. This UTF-8 CSV has a header and one row per query:

```text
run_id,repetition,query_id,returned,matched,ground_truth_count,latency_us,latency_ms,recall,physical_reads,page_server_reads,hot,event_name
```

`recall` is a fraction from 0 to 1, and `latency_ms` is `latency_us / 1000.0`.
Unavailable counters and unknown `hot` values are blank, not zero; known `hot`
values are `True` or `False`. Successful short and empty answers are included.
Warm-up passes and failed evidence validation do not produce measured CSVs.
Existing CSVs are never overwritten. A CSV export failure marks the repetition
failed and retains its JSON and evidence without replaying any search; a partial
CSV may remain and must not be treated as complete. The existing JSON, settings,
and XEvent evidence remain unchanged. CSV export uses Python's standard library.

## Local Tests

From the repository root:

```powershell
python -m unittest discover -s azure-sql-vector-performance/tests -v
```

The tests use tiny binary fixtures served over local HTTP. They cover all three
size selections, exact saved bytes, manifest checksums, skipped files, partial
files, progress, and failures. They do not download the real
datasets, connect to SQL, or prove supplied ground-truth accuracy.

The load tests use small fixtures and mocked tool calls to check multiple chunks,
partial final chunks, continuous source IDs, native bytes, source validation,
tool checks, independent transaction size, and success-only deletion. They cover
BCP failures, zero-exit rejected rows, and failed SQL count verification. These
tests do not establish that SQL syntax or installed tools work with a live server.

[tests/test_live_load.py](tests/test_live_load.py) is skipped unless
`MSSQL_VECTOR_LIVE_TEST=1`. It requires `MSSQL_VECTOR_TEST_SERVER`,
`MSSQL_VECTOR_TEST_DATABASE`, `MSSQL_VECTOR_TEST_USERNAME`, `MSSQL_VECTOR_TEST_BCP`,
and `MSSQL_VECTOR_TEST_SQLCMD`; optionally set
`MSSQL_VECTOR_TEST_TRUST_SERVER_CERTIFICATE=1` for a local test server. Run it only
with explicit approval against an existing dedicated database without the three
target tables. It overrides expected corpus/query counts only inside the test,
imports three 1,280D documents, two queries, and six GT rows across chunks, then
compares every exported fixture byte against independent expectations. This is
tiny-fixture coverage, not a full 1M load or a sample of the downloaded corpus.
It uses hidden SQL-password prompts and retains its artifacts and tables without
automatic cleanup. It has not been run for this refactor.

[tests/test_build.py](tests/test_build.py) mocks pyodbc connections/cursors and
checks one-connection execution order, parameter binding, timer placement, TLS
defaults, optional version pinning, and existing-index refusal. It also covers
replacement confirmation, late result-set errors, interruption, credential
redaction, and original-exception preservation when export/cleanup fail. These
are not live SQL syntax, CREATE, or elapsed-time validation. No index build has
been executed by these tests.

[tests/test_search.py](tests/test_search.py) covers stamp decoding, event-field
filtering, missing/duplicate/lost/truncated evidence, exact attribution, microsecond
units, short/empty recall, percentiles, cache counters, result-set draining,
evidence-before-stop ordering, credential handling, and one warm-up followed by
ten passes using mocked connections. Install the search requirements before
running the full suite. The approved 10-query live attempt on 2026-09-24 reached
preflight on build 18.0.251.0 but stopped because the document table had only its
primary-key index, not the required vector index. No warm-up, searches, or XEvent
collection were started. The failure is retained under
`data/search-runs/1M-small-20260924-135124`. Workload SQL syntax, actual event
delivery, and a successful small end-to-end result still require validation after
the index and selected GT data are available, before scaling.