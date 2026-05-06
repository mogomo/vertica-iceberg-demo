# vertica-iceberg-demo

A small, version-aware Vertica × Apache Iceberg demo you can run end-to-end in
about a minute.

## What this is

A self-contained demo that walks through the three statements on Vertica's
[`CREATE EXTERNAL TABLE ICEBERG`](https://docs.vertica.com/latest/en/sql-reference/statements/create-statements/create-external-table-iceberg/)
documentation page (`IcebergPathMapping`, `CREATE EXTERNAL TABLE ICEBERG`, and
the `COLUMN TYPES` override) against a real Iceberg table. On Vertica 25.4 or
later, it also runs an `EXPORT TO ICEBERG` round-trip first, so the read demo
points at a table Vertica itself just wrote.

The demo:

- Detects your Vertica version automatically.
- Detects your storage backend (S3/MinIO, HDFS, or local filesystem) from the
  URL scheme of `ICEBERG_DIRECTORY` in your `.env`.
- Prints which Iceberg features your version supports, sourced from the
  official documentation.
- Decides whether to run the write demo (v25.4+) or use bundled sample files
  (older versions), and tells you exactly what to do if anything is missing.

## Quick start

You need a Vertica cluster you can reach with `vsql`, plus an S3-compatible
object store, HDFS, or shared filesystem path readable from every Vertica node.

```bash
git clone https://github.com/<your-handle>/vertica-iceberg-demo.git
cd vertica-iceberg-demo

cp env.example .env
chmod 600 .env
$EDITOR .env                   # set MINIO_ACCESS_KEY, MINIO_SECRET_KEY, etc.

./run.sh
```

That's it. On Vertica 25.4 or later, the script writes a fresh Iceberg table
and reads it back. On older versions, the script tells you exactly which line
to add to `.env` and which `mc cp` (or `hdfs dfs -put`) command to run; the
next invocation succeeds.

## Files

```
README.md                   this file
LICENSE                     MIT
env.example                 config template — copy to .env and edit
.gitignore                  keeps .env, rendered SQL, etc. out of git
run.sh                      orchestrator: detects, decides, dispatches

01_setup_session.sql.tmpl   ALTER SESSION setup
02_demo_read.sql.tmpl       Statements 1, 2, 3 from the docs (read path)
03_demo_write.sql.tmpl      EXPORT TO ICEBERG round-trip (v25.4+)

samples/sales/              pre-built 3-row Iceberg table
  data/<...>.parquet
  metadata/v1.metadata.json
  metadata/<...>.avro
```

## Configuration

All settings live in `.env`. The template (`env.example`) documents every
option, but the minimum is:

| Variable | Example | Notes |
| :--- | :--- | :--- |
| `VSQL_USER` | `dbadmin` | who connects |
| `VSQL_HOST` | `n01` | Vertica node |
| `VSQL_PORT` | `5433` | Vertica port |
| `VSQL_DB` | (optional) | needed only with multiple databases |
| `ICEBERG_DIRECTORY` | `s3://iceberg-demo/sales` | where the Iceberg table lives or will be written |
| `MINIO_ENDPOINT_HOST` | `n06` | MinIO/S3 host (S3-style only) |
| `MINIO_ENDPOINT_PORT` | `9000` | MinIO/S3 port |
| `MINIO_USE_HTTPS` | `0` | `0`=HTTP, `1`=HTTPS |
| `MINIO_ACCESS_KEY` | your key | required for S3-style |
| `MINIO_SECRET_KEY` | your secret | required for S3-style |
| `MINIO_REGION` | `us-east-1` | required by SDK; ignored by MinIO |
| `ICEBERG_METADATA` (optional) | `s3://.../v1.metadata.json` | point at an existing Iceberg table |

The script enforces `chmod 600` on `.env` and shreds rendered SQL files on
exit, so credentials never persist in world-readable form.

## Storage backends

The script auto-detects from `ICEBERG_DIRECTORY`:

| Scheme | Detected as | Required `.env` variables |
| :--- | :--- | :--- |
| `s3://...` | minio / s3 | `MINIO_*` set |
| `hdfs://...` | HDFS | none — Vertica reaches HDFS via its `HadoopConfDir` |
| `/...` | local | none — must be readable from every Vertica node |

For HDFS and local filesystem paths, the script automatically skips the AWS
session parameters (the `ALTER SESSION SET AWS*` lines are commented out
during template rendering).

## When `ICEBERG_METADATA` is unset

This is where the demo's behaviour depends on your Vertica version:

- **v25.4+** — the script runs `EXPORT TO ICEBERG` to land a fresh Iceberg
  table at `ICEBERG_DIRECTORY`, then reads it back. No further action needed.

- **v11.1 to v25.3** — `EXPORT TO ICEBERG` did not exist yet. The script
  prints a `Missing:` block followed by a `Fix example:` block showing exactly
  how to copy the bundled `samples/sales/` to your storage and which line to
  add to `.env`. Three lines, then rerun.

Example output on an older version:

```
Missing: ICEBERG_METADATA in .env
Fix example:
  ./bin/mc cp -r samples/sales/ localminio/iceberg-demo/sales/
  then add this line to .env:
    ICEBERG_METADATA=s3://iceberg-demo/sales/metadata/v1.metadata.json
```

## Bundled samples

`samples/sales/` is a real, working 3-row Iceberg table:

| order_id | product     | amount | city        |
| :---     | :---        | :---   | :---        |
| 1        | widget      | 19.99  | Springfield |
| 2        | gadget      | 29.50  | Shelbyville |
| 3        | thingamajig |  5.25  | London      |

Schema: `BIGINT`, `string` → `VARCHAR(80)`, `decimal(10,2)` → `NUMERIC(10,2)`,
`string` → `VARCHAR(80)`.

The sample's metadata embeds `s3://iceberg-demo/sales/` as the table location.
If your `ICEBERG_DIRECTORY` is different, the demo's first statement
(`IcebergPathMapping`) automatically bridges the gap at query time. The demo's
Statement 1 is therefore not just a syntax illustration — it does real work
whenever your storage layout differs from the embedded prefix.

## Compatibility

Tested on:

- **Vertica 26.1.0-0** with MinIO on a separate node, plain HTTP — full read
  and write demo passes.

The SQL templates are structured per the published documentation for each
release from v11.1 onwards, but live-cluster runs against older versions have
not been performed. If you run it on another release, an issue or pull
request with your output is very welcome.

The bundled samples were generated with PyIceberg 0.11 and use Iceberg
format-version-2 metadata, which is documented as supported from Vertica 24.1
onwards. On v23.x clusters where only v1 metadata is supported, supply your
own v1-format `ICEBERG_METADATA` instead of using the bundled samples.

## Troubleshooting

| Symptom | Likely cause | Fix |
| :--- | :--- | :--- |
| `ERROR: .env permissions are 644` | script enforces mode 600 | `chmod 600 .env` |
| `Cannot stat metadata file ...` | path mismatch, missing credentials, or unreachable storage | check `ICEBERG_DIRECTORY`, MinIO host, credentials |
| `Problem reading metadata for table ...` | metadata format incompatibility (e.g. v2 on Vertica < 24.1) | use your own v1 metadata via `ICEBERG_METADATA`, or upgrade |
| Hangs on Statement 2 | session credentials lost between SQL files | already handled in current `run.sh` |
| `Could not determine type of column ...` | type override clashes with Iceberg type | check the `COLUMN TYPES` clause syntax |

## Contributing

Issues and pull requests are welcome — particularly verified runs on Vertica
versions other than 26.1, additional storage backends, or improvements to
the troubleshooting table.

## License

MIT.

```
MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

The same text is also expected in a separate `LICENSE` file at the repo root,
which GitHub uses for license auto-detection.

## Disclaimer

This software is provided **"as is", without warranty of any kind**, express
or implied. The author is not affiliated with OpenText or with the Vertica
product team, and this demo is not an official OpenText or Vertica
deliverable. Use at your own risk on non-production data only until you have
verified its behaviour in your environment.

Apache Iceberg is a trademark of the Apache Software Foundation. Vertica and
OpenText are trademarks of Open Text Corporation. References to these
products in this repository are made for descriptive purposes only.
