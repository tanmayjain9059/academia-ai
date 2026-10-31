# Deployment and cost strategy

Status: candidate deployment plan. Prices and quotas are snapshots, not guarantees; re-check before procurement and launch.

## 1. Deployment principles

- Public web app, private data plane.
- Durable PostgreSQL and object storage; no reliance on ephemeral app disks.
- Separate frontend, API, CPU workers and model inference capacity.
- Provider-independent inference adapter.
- Staging and production environments with separate secrets/data.
- Spend limits, quotas and alerts before opening public registration.

## 2. Hugging Face role

Use Hugging Face for model discovery, evaluation, demos and potentially paid inference hosting. Do not make the production service dependent on unlimited free compute.

Current official references:
- Spaces overview: https://huggingface.co/docs/hub/en/spaces-overview
- GPU Spaces and billing: https://huggingface.co/docs/hub/en/spaces-gpus
- Inference Providers pricing: https://huggingface.co/docs/inference-providers/pricing

The current Spaces overview lists CPU Basic at 2 vCPU, 16 GB RAM and non-persistent disk; paid GPU choices are billed hourly. Inference Providers are pay-as-you-go; current documentation lists $0.10 monthly credits for free users, subject to change. These facts rule out treating a free account as production-scale inference capacity.

## 3. Candidate hosting roles

- Frontend: static hosting/CDN.
- API and CPU workers: container platform with persistent network access to Postgres, object storage and queue.
- PostgreSQL: managed Postgres with backups, pooling and pgvector support verified. Supabase is a concrete evaluation candidate because its current region list includes Mumbai (AWS ap-south-1); verify exact plan capabilities and data-processing scope before choosing.
- Object storage: Cloudflare R2 or equivalent private S3-compatible service. R2 supports S3-compatible access and presigned PUT/GET/HEAD/DELETE; signed form POST is not supported. Use short-lived, object-scoped URLs and validate uploads again server-side.
- Queue: durable managed queue with visibility/lease extension, bounded retries and dead-letter handling. Evaluate SQS, managed Redis worker queues and QStash rather than assuming all queue products have identical parallelism/delivery semantics.
- Inference: benchmarked GPU endpoint or provider API with explicit data-processing terms and cost tracking.

Cloudflare R2 pricing documentation currently states no internet egress charge and lists standard storage at $0.015/GB-month, with operation charges and monthly free allowances described on its pricing page: https://developers.cloudflare.com/r2/pricing/

Neon's free-tier documentation currently lists 100 CU-hours/project, 1 GB storage/project and scale-to-zero behavior. This may be useful for development, but free quotas are not a production availability commitment: https://neon.com/docs/introduction/pro-plan

## 4. Prototype vs production

### Prototype
- Low traffic, invite-only.
- Free/low-cost frontend and database tiers where suitable.
- Private object storage with strict size caps.
- CPU parsing and limited inference calls.
- Manual or hard quotas for expensive OCR/LLM work.
- Clear "demo" availability expectations.

### Public beta
- Paid managed database with backups and connection pooling.
- Durable object storage.
- API and worker instances with health checks.
- Paid inference budget or self-hosted GPU with measured utilization.
- Rate limits, per-user quotas, billing alerts and error monitoring.
- Data deletion/export and incident response process.

### Production
- Availability targets and recovery objectives.
- Capacity planning from measured concurrency and page/answer costs.
- Multiple worker pools and controlled GPU scaling.
- Provider failover where it does not compromise data handling.
- Security review, backup restore drills and load tests.

## 5. Cost model

Track cost per:
- Uploaded GB and retained GB-month.
- Parsed page.
- OCR page by model route.
- Embedded chunk.
- Reranked query.
- Generated answer by model, input/output tokens and retry.
- Exported PDF.
- Active tenant/month.

Model monthly cost as:
storage + database + API/worker compute + queue + inference + observability + email/auth + egress/operations.

Never promise "free forever." A free tier is a development allowance, not a scaling strategy. The product should support quotas, usage metering, plan limits and a paid inference path before public growth.

## 6. Provider selection gates

Score candidate providers on:
- Region and data processing terms.
- GPU type, memory and model compatibility.
- Cold-start time and uptime.
- Throughput under concurrent requests.
- Queueing/batching support.
- Persistent storage behavior.
- Logs and metrics.
- Hourly/request pricing and idle billing.
- Ability to export/migrate data and inference configuration.

No provider is selected as final until a representative workload benchmark and owner cost ceiling are agreed.

See [Technical research and provisional decisions](TECHNICAL_DECISIONS.md) for dated evidence on Postgres, object storage, queue semantics, auth and parser licensing.
