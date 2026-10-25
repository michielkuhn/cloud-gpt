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
48
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
| IaaS VM | <!--answer:vm_cold_ms-->253.9<!--/answer--> | <!--answer:vm_warm_median_ms-->210.6<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1330.3<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->634.8<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1487.8<!--/answer--> | <!--answer:function_warm_median_ms-->636.9<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
vm: warm, n=20, median 210.6 ms, cold 253.9 ms
   200-250  ms | #################### 20

cloudrun: warm, n=20, median 634.8 ms, cold 1330.3 ms
   100-150  ms | ## 2
   600-650  ms | ################# 17
   650-700  ms | # 1

function: warm, n=20, median 636.9 ms, cold 1487.8 ms
   100-150  ms | ### 3
   600-650  ms | ############ 12
   650-700  ms | ##### 5
```
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
The VM creates nothing under load. Every request lands on one e2-micro running a single gunicorn process with 8 threads, so ten requests share those threads and a hundred simply queue. Its hard limit is the machine type I picked (e2-micro: 2 shared vCPUs, 1 GB RAM). Cloud Run starts additional container instances from my image as load rises, each serving up to 80 concurrent requests, capped by my --max-instances=3. The Function also adds Cloud Run instances, but with a concurrency of 1 every simultaneous request needs its own instance, so a burst of 100 creates up to 100 instances, the default maximum since I set no cap.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM was by far the most work: I built a custom VPC, a subnet and two tagged firewall rules, granted the VM's service account read access to Artifact Registry, created the instance, then SSHed in to install Docker, authenticate it and start the container. From then on I own OS patching, container restarts and a machine billed around the clock. Cloud Run needed only the image I had already built and one deploy command, with HTTPS, a public URL, health checks and autoscaling included. The Function needed the least: no Dockerfile and no image, just source code, requirements.txt and an entry point, with Google building the container for me during the deploy.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-784075097191.us-central1.run.app
LAST DEPLOYED BY: service-784075097191@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-25T14:30:24.494880Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-784075097191.us-central1.run.app
LAST DEPLOYED BY: michiel-nicholaas.kuhn@eurecom.fr
LAST DEPLOYED AT: 2026-10-25T14:28:42.574374Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The list shows two Cloud Run services: echo-bot, deployed by me, and echo, deployed by gcf-admin-robot, the Cloud Functions service agent. My gen2 Function is therefore itself a Cloud Run service. gcloud functions deploy uploaded my source, Cloud Build turned it into a container image using Google's Python buildpack and the Functions Framework, stored it in the managed gcf-artifacts repository, and deployed it to Cloud Run. Both paths share the same scale-to-zero infrastructure, front end and cold-start mechanics, which is why the cold times (1330.3 vs 1487.8 ms) and warm medians (634.8 vs 636.9 ms) nearly match. What genuinely differs is who builds the image (my multi-stage Dockerfile and gunicorn versus a buildpack), the defaults (concurrency 1, 256 MB and 1/6 vCPU for the Function versus 80, 512 MiB and 1 vCPU for Cloud Run), and the Function's extra layer for event triggers and the cloudfunctions.net URL.
<!--/answer-->


---

## Additional notes

**VM cold start.** The VM has no meaningful cold start: its container runs continuously, so the first
request measured by measure.py hits an already-warm process. The small gap between its cold and warm
numbers is first-connection overhead, not a boot. The price of that is paying for the machine 24/7.

**Why the VM is faster when warm.** Measurements were taken from Cloud Shell, which runs near me in Europe,
against us-central1. The VM serves plain HTTP directly, so a request costs roughly two round trips (TCP
handshake plus request), matching its very flat ~211 ms. Cloud Run and the Function are reached over HTTPS
through Google's front end, and measure.py opens a new connection per request, so each request also pays
for TLS setup and the extra hop. The few ~120 ms samples on both serverless platforms show the service itself
responds quickly; most of the difference is network and connection setup rather than compute.

**Network and firewall tag.** The VM runs in the custom-mode VPC echo-net, subnet echo-subnet
(10.10.0.0/24); its internal IP 10.10.0.2 confirms the placement. A custom VPC starts with only implied
allow-all-egress and deny-all-ingress, so I added allow-echo-8080 (TCP 8080) and allow-echo-ssh (TCP 22).
Both use --target-tags=echo-http, so they apply only to instances carrying that tag. Without the tag the
rules would apply to every instance in the network, exposing anything later launched there to the internet;
the tag scopes exposure to exactly the machines meant to serve the echo bot.
