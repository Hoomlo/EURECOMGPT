# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->52.9<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.037<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->5.1<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->15<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
From fastest to slowest: local single-process word count (0.037 s), PySpark TF-IDF (5.1 s), and cloud MapReduce word count (52.9 s). Only the local and cloud word counts are like-for-like. The local run computes purely in memory with zero network or scheduling overhead. Spark executes a heavier TF-IDF pipeline (two reduceByKeys and a join) in 5.1 s, mostly spent on JVM/task scheduling and in-memory/local shuffle. Cloud MapReduce takes 52.9 s because of per-task cloud overhead: HTTP function invocations, Cloud Storage round trips, workflow orchestration, and chaos retries on a small corpus.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
With 4 stages, the Spark job had 3 shuffle boundaries, caused by reduceByKey in term_freq, reduceByKey in doc_freq, and join in build_tfidf_rdd. With 15 retries across 16 tasks (15 out of 31 total attempts, or ~48% of attempts), the failure rate is slightly higher than the expected 30% from --chaos 0.3 due to small-sample variance.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
In spark_tfidf.py and the notebook, parallelize, flatMap, reduceByKey (in term_freq and doc_freq), map (keying by term and to_row), and join are all lazy transformations, while collect() is the only action. Calling build_tfidf_rdd executed no computation and only constructed the logical lineage DAG; calling collect() triggered Spark to materialize the stages, run tasks, and return the 5,975 rows.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
If a worker holding one partition of the first reduceByKey (term_freq, Stage 0) died, Spark would recompute only that lost partition starting from ParallelCollectionRDD through flatMap and the Stage 0 shuffle boundary. It could reuse the surviving partitions of Stage 0 and all already-materialized shuffle outputs from unaffected partitions rather than restarting the whole job.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows with mr-wordcount<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run with mr-worker<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->mapreduce.py<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is provided by the parallel: for: block of map_phase in workflow.yaml: Cloud Workflows waits for all parallel map iterations (including any retries) to finish before advancing to reduce_phase.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Retrying a failed task is safe because workers are stateless and write their output objects atomically to deterministic Cloud Storage paths, overwriting any partial or prior output from an earlier attempt. If run_map appended pairs instead of overwriting, a retried task (or duplicate execution) would write its key-value pairs twice into the shuffle file, inflating the final word counts.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
During the --chaos 0.3 run (832595a3-1944-4a15-8f39-ca433d9b1e51), Cloud Run revision logs recorded 15 ERROR responses from injected worker crashes alongside container autoscaling startups while Workflows retried the failed steps. Despite the failures, the workflow finished with state SUCCEEDED in 52.9 s and produced results identical (matches_local: true) to the local reference.
<!--/answer-->
