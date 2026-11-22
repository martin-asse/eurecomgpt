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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->35.3<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.054<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.5<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->8<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Fastest to slowest: local single process (0.054 s), Spark (4.5 s), cloud MapReduce (35.3 s). The like-for-like pair is local vs cloud: the same word count over the same 60 documents (about 7.7k pairs), so the gap of roughly 650x is pure overhead. Each cloud task pays a Cloud Run invocation (with cold starts), Cloud Storage reads and writes for the shuffle, and workflow scheduling plus retry backoff (this was the --chaos 0.3 run; the clean run took 30.0 s). Spark does heavier work (two reduceByKeys and a join) in 4.5 s on one machine, so the cloud job's time is dominated by coordination, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark job had 4 stages, separated by 3 shuffles: the reduceByKey in term_freq, the reduceByKey in doc_freq, and the join in build_tfidf_rdd. flatMap and map are narrow and cause no shuffle. In the chaos run, 8 attempts failed out of 24 total (16 tasks plus 8 retries), about 33%, close to the 30% injected (about 7 expected).
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
flatMap, map, reduceByKey and join are transformations: they are lazy and only extend the lineage. collect is the only action. Running build_tfidf_rdd returned immediately and computed nothing; it just built the lineage, which toDebugString printed. Running collect triggered the whole pipeline (stages, shuffles, tasks) in 4.5 s and returned the 5975 TF-IDF rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Spark would recompute only the lost partition, following the lineage back to the first shuffle boundary (the term_freq reduceByKey): it re-reads the map-side shuffle files of stage 0, and re-runs those flatMap map tasks only if the dead worker held them. All other partitions and downstream stages already computed are reused, so nothing restarts from scratch.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows, running workflow.yaml (the mr-wordcount workflow)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The mr-worker Cloud Run function (main.py), one invocation per map or reduce task<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->The Cloud Storage bucket, under the per-job prefix (input/, shuffle/ and output/)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition() in mapreduce.py: zlib.crc32(word) % num_reducers<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The parallel step of map_phase: it only completes once all its for-branches have finished, so reduce_phase, the next step in the workflow, cannot start before the slowest mapper is done.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
A task reads fixed inputs and writes to deterministic object names (per job, split and partition), so re-running it overwrites any partial output and yields the same result, however many attempts happen. If run_map appended, a retry after a partial write, or after a timeout where the first attempt actually finished, would add its pairs twice, inflating the counts so they no longer match the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
The execution still ended SUCCEEDED (35.3 s vs 30.0 s without chaos) and the output was identical to the local reference (matches_local: true, 2221 distinct words). In Logs Explorer I counted 8 mr-worker requests with status 503 (the simulated worker crashes); the workflow retried each with exponential backoff, and no task used up its 5 retries.
<!--/answer-->