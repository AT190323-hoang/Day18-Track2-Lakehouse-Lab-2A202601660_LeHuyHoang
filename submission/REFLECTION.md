# Reflection — Day 18 Lakehouse Lab

**Le Huy Hoang — MSSV 2A202601660**

The anti-pattern our team's data is most at risk of: **the small-file
problem from streaming ingestion with no compaction job** — not a bug so
much as the absence of a cron entry.

NB6 reproduced it directly: 200 micro-batches (5-second-trigger-style
ingestion) left 200 files averaging 51.5 KB each, nowhere near the
128–512 MB production target. At 50K queries/day, a full scan over those
200 files costs ~$4.00/day in S3 GET requests alone versus $0.08/day for
the same data in 4 compacted files — a 50× difference driven purely by
object count, not data volume.

Our real risk: any pipeline with a short micro-batch trigger
(Kafka→lakehouse, 5–10s windows) accumulates this silently. Every
individual commit is correct; only the accumulation is the bug, and it
stays invisible until query latency or the storage bill spikes. Unlike a
code bug, nothing fails loudly — no exception to catch, no test that says
"you forgot the compaction cron." The fix has to be operational (a
scheduled `OPTIMIZE` / `rewrite_data_files` job), not a code-review
checkpoint — exactly the failure mode that is easy to miss until it is
already expensive.
