# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
47 MB
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->241.0<!--/answer--> | <!--answer:vm_warm_median_ms-->212.1<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2246.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->631.0<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1672.3<!--/answer--> | <!--answer:function_warm_median_ms-->636.5<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
[OK  ] vm        cold=  241.0ms  warm_median=  212.1ms
            #############
[OK  ] cloudrun  cold= 2246.7ms  warm_median=  631.0ms
            ########################################
[OK  ] function  cold= 1672.3ms  warm_median=  636.5ms
            ########################################
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The IaaS VM creates nothing automatically; load is handled solely by the single e2-micro instance, limited by its static hardware capacity. Cloud Run and Cloud Function automatically spin up new container instances to absorb concurrent requests. Their scaling is strictly capped at a maximum of 3 instances (--max-instances=3) as configured during deployment, whereas the VM is fundamentally constrained by its vCPU and memory resource limits.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
For the IaaS VM, I manually built a custom VPC network, subnets, firewall rules, provisioned an e2-micro instance, installed Docker, and configured IAM credentials. 
For Cloud Run, we containerized the app via a multi-stage Dockerfile and pushed it to Artifact Registry before deploying a managed service. 
For the Cloud Function, deployment was the simplest: we provided raw source code directly via --source=., letting Google Cloud handle the buildpack and serverless infrastructure provisioning automatically.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-61000093214.us-central1.run.app
LAST DEPLOYED BY: service-61000093214@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-27T17:38:11.674728Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-61000093214.us-central1.run.app
LAST DEPLOYED BY: teo.altmeyer@eurecom.fr
LAST DEPLOYED AT: 2026-10-27T17:05:21.608801Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The gcloud run services list output reveals that the Cloud Function runs natively as a managed Cloud Run service under the hood. 
This explains why their cold starts and warm latencies are remarkably similar. 
Both execute containerized workloads using identical serverless infrastructure that scales to zero. 
The fundamental difference lies in the packaging workflow: Cloud Run requires explicit containerization, a Dockerfile, and an Artifact Registry image, whereas the Cloud Function abstracts this away by accepting raw source code and building the container automatically via Google Cloud buildpacks.
<!--/answer-->
