# ADR-001. AWS-only service mapping for the omi fork

- Status: Proposed
- Date: 2026-11-23
- Deciders: fork owner (sauhsoj), on kanban initiative `task-1795234744-1fd1cb759127900eddba81-76dd`
- Owner: omi-aws initiative implementation cards
- Review by: 2027-02-23, or when the first implementation card lands, whichever is first

This ADR applies to the `jsamuel1/omi` fork only. Upstream BasedHardware/omi
keeps its GCP stack. It is the design input for the implementation cards that
port the MVP loop (sign-in, device and firmware, live transcription,
conversations, memories, chat) to AWS. The route-by-route census is in
[MVP-API-SURFACE.md](MVP-API-SURFACE.md). Line references are to the
`origin/main` tree at `a04153d180`. Anything not verified against code, docs or
the AWS account is marked `UNVERIFIED`.

## Context

- **Target.** Account `717494614307` (profile `sauhsoj+ct-pihole-Admin`), region
  `ap-southeast-2`, public host `omi.home.sauhsoj.wtf` in Route53 zone
  `Z01865332JZ618N8XF2YM`. That zone has no `omi` records today. One user at
  launch, so cost floors matter more than throughput.
- **Identity.** The existing Cognito user pool is `ap-southeast-4_jA7oAYYrp`
  ("home-portal-users", Melbourne), with hosted UI at `auth.home.sauhsoj.wtf`. It
  is defined in `~/src/sauhsoj-account-portal/lib/constructs/cognito.ts`. The
  initiative brief named `ap-southeast-2_PBPhR6ziD`. That pool is "pihole-auth",
  a different pool, and the owner confirmed the Melbourne pool on 2026-11-23.
- **Firestore is the whole data layer.** 270 production files import
  `google.cloud.firestore`, and `backend/database/` holds 361 modules. Measured
  usage: 460 `firestore.transactional`, 102 `Increment`, 101 `DELETE_FIELD`,
  73 `FieldFilter`, 15 `ArrayUnion`, 8 `SERVER_TIMESTAMP`, 33 `.count().get(`,
  8 `.sum(` aggregations, `collection_group` in 21 files, and `AsyncClient(` in 38
  places (`database/conversations.py`, `database/knowledge_graph.py`).
  `firestore.indexes.json` declares 271 composite indexes and 31 field overrides.
  No `on_snapshot` listener sits on the MVP path.
- **Ports and cells are partial.** `database/ports/` covers `action_items`,
  `folders`, `goals`, `people` and `workstreams`. `database/cells/` is a Postgres
  CellStore (`CELLS_PG_DSN`, migrations `0001_cells.sql` to
  `0004_task_workflow_typed.sql`). Neither covers users, conversations, memories,
  messages or chat sessions, which are the MVP collections.
- **Postgres already runs in production.** CellStore and the jobstore
  (`JOBSTORE_PG_HOST`/`_DATABASE`/`_USER`, `database/jobstore/dual_store.py:722`)
  use psycopg 3.2.10.
- **Auth is Firebase ID tokens.** `utils/other/endpoints.py::verify_token` (line
  143) calls `firebase_admin.auth.verify_id_token`. `dependencies.py:72` does the
  same. Firebase exception and verify references appear 25 times across 8 files.
  `main.py:86-104` and `pusher/main.py:36-39` call
  `firebase_admin.initialize_app` at import.

## Decision

### Service mapping

One chosen AWS replacement per dependency on the MVP path. "Ports/cells" says
whether `database/ports/` or `database/cells/` already provides the seam.

| GCP / third-party dependency | AWS replacement (chosen) | Rejected alternative (reason) | Swap seam | Ports/cells |
| --- | --- | --- | --- | --- |
| Firestore (document store, transactions, queries, aggregations) | Aurora Serverless v2 PostgreSQL 16, accessed through a Firestore GAPIC-surface adapter over `jsonb` tables | DynamoDB (100-item transaction cap, no interactive read-then-write transactions, 20 GSI cap against 271 indexes, 400 KB items) | `database/_client.py::_build_firestore_client` (line 212) builds `firestore.Client` with an injected `_firestore_api` | No. Ports/cells cover 5 non-MVP domains only |
| Firebase Auth (ID token verify) | Cognito access token, verified with PyJWT 2.15.1 `PyJWKClient` against the pool JWKS | Cognito ID token (carries no `client_id` claim and is meant for the client, not resource servers) | `utils/other/endpoints.py::verify_token` (143), `_verify_ws_auth` (390), `dependencies.py::get_current_user_id` (64) | No |
| Firebase Admin user lookup | `cognito-idp:AdminGetUser(Username=sub)` | `ListUsers` with a `sub` filter (extra call, eventually consistent) | `database/auth.py::_firebase_get_user` (line 13, 8 call sites) | No |
| Firebase custom-token sign-in (`/v1/auth/*`, Google and Apple) | Cut. Cognito hosted UI with PKCE replaces it | Federating Google and Apple IdPs into the pool now (the pool has COGNITO as its only IdP, and the owner admin-creates users) | `routers/auth.py` stops being called by the client | No |
| GCS buckets | S3, one bucket per `BUCKET_*` env var, behind a new `S3StorageClient` | One shared bucket with prefixes (breaks per-bucket lifecycle and IAM scoping that the env-var split already implies) | `utils/other/local_storage.py::create_storage_client` (line 348), plus direct `google.cloud.storage` importers listed below | No |
| Cloud Tasks (finalization dispatch) | Inline dispatch to a pusher sidecar for MVP. SQS is the post-MVP queue | Step Functions (heavier than a FIFO work queue for one user) | `utils/pusher.py` (`HOSTED_PUSHER_API_URL`), `routers/listen/conversations.py::schedule_finalization` (330) | No |
| Memorystore / Redis Cloud | ElastiCache for Valkey, node-based, `cache.t4g.micro`, cluster mode disabled | ElastiCache Serverless (cluster-mode only; the code uses 19 `r.eval`, 12 `register_script`, 6 pipelines and 2 `mget`, which risk CROSSSLOT) | `database/redis_connection.py::redis_connection_kwargs` (line 9) gains an `ssl` kwarg | No |
| Soniox / Modulate streaming STT | Amazon Transcribe streaming, as a new `STTService.transcribe` and `TranscribeStreamingSocket(STTSocket)` | Self-hosted Parakeet on GPU (GPU cost floor for one user) | `utils/stt/streaming.py::STTService` (122), `config/stt_provider_policy.py::DEFAULT_MODELS_BY_SURFACE` (145) | No |
| OpenAI / OpenRouter / Anthropic LLMs | Amazon Bedrock through `langchain-aws` `ChatBedrockConverse`, `au.` inference profiles | Bedrock's OpenAI-compatible endpoint (serves gpt-oss only, no Claude) | `utils/llm/providers.py` (new `bedrock` provider), `utils/llm/clients.py::get_llm` (762), `utils/llm/model_config.py` (new `MODEL_QOS` profile) | No |
| OpenAI embeddings | Bedrock `amazon.titan-embed-text-v2:0` (1024-dim) via `BedrockEmbeddings` | Cohere Embed on Bedrock (no gain at this scale, one more model to approve) | `utils/llm/clients.py::_OpenAIEmbeddingsProxy` (356) | No |
| Pinecone (conversation vectors) | Cut for MVP. pgvector on the same Aurora cluster later | OpenSearch Serverless (OCU minimum is a standing cost for one user) | `database/vector_db.py` (`PINECONE_API_KEY`/`PINECONE_INDEX_NAME` at 144-155) | No |
| Typesense (conversation search) | Cut for MVP. Postgres full-text search later | OpenSearch managed domain (standing instance cost) | `utils/conversations/search.py::search_conversations` (342) | No |
| Firebase Cloud Messaging push | Cut for MVP. SNS mobile push to APNs later | Amazon Pinpoint (end of support 2026-10-30) | New `OMI_PUSH_PROVIDER` seam in `utils/notifications.py` | No |
| GitHub Releases (firmware manifest) | Kept as is. Not GCP | S3 mirror of firmware binaries (duplicates a public source that already has a Redis LKG cache) | `utils/github_releases.py` | n/a |
| Secret Manager / env files | AWS Secrets Manager, injected as ECS task secrets | SSM Parameter Store SecureString (no managed rotation for the RDS secret) | ECS task definition `secrets` | n/a |
| GKE / Cloud Run hosting | ECS Fargate behind an ALB | App Runner (no WebSocket support) | Container image from `backend/Dockerfile` and `backend/pusher/Dockerfile` | n/a |
| Cloud KMS | Not on the MVP path. `utils/other/space_keys.py` serves the object lake only | n/a | n/a | n/a |

Direct `google.cloud.storage` importers that must route through the storage
client: `utils/batch_pipeline/record_gap_fill.py`, `utils/batch_pipeline/artifacts.py`,
`utils/other/audio_chunks.py`, `routers/listen/parity_pack_export.py`.

### Document store: Aurora Serverless v2 PostgreSQL, not DynamoDB

The decision rests on how the code uses Firestore, not on the data volume.

- **Transactions.** 460 `@firestore.transactional` functions read, decide and
  then write. DynamoDB `TransactWriteItems` is write-only with condition
  expressions, capped at 100 items, so each of those functions would need
  redesign. Postgres runs them as ordinary `SERIALIZABLE` or `SELECT ... FOR
  UPDATE` transactions.
- **Queries and indexes.** 271 composite indexes, `collection_group` queries in
  21 files, and `.count()`/`.sum()` aggregations map to B-tree and expression
  indexes on `jsonb` plus SQL aggregates. DynamoDB caps a table at 20 GSIs and has
  no server-side count or sum.
- **Existing SQL.** CellStore and the jobstore already run on Postgres through
  psycopg 3.2.10, so one engine serves all three.
- **Cost mechanism.** Aurora Serverless v2 bills ACU-hours plus storage and I/O.
  A minimum capacity of 0 ACU enables auto-pause (`SecondsUntilAutoPause`
  300-86400). Resume takes about 15 s, or 30 s or more after more than 24 h
  paused ([auto-pause docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html)).
  The backend runs background reconcile loops (`main.py` imports
  `reconcile_listen_finalization_jobs` and three siblings) that may keep the
  cluster from pausing. Minimum capacity is therefore a tuning lever: start at
  0.5 ACU and measure. Pricing:
  [Aurora pricing](https://aws.amazon.com/rds/aurora/pricing/). DynamoDB
  on-demand would be cheaper at idle, which does not outweigh the rewrite.

**Adapter shape.** Keep the real `google-cloud-firestore` SDK classes and
replace only the transport. The adapter implements the GAPIC RPCs the SDK calls
on `client._firestore_api`: `batch_get_documents`, `run_query`,
`run_aggregation_query`, `commit` (with field transforms and preconditions),
`begin_transaction`, `rollback`, `list_documents` and `list_collection_ids`.
`tests/unit/fixtures/offline_firestore_sdk.py` already proves this injection
pattern. Keeping the SDK classes keeps the class-level patches working:
`batch_privacy_write_fence.install_write_fence`, `memory_change_head.install_write_heads`,
`list_projection`, `firestore_write_probe` and `firestore_document_probe`.

Table layout: one `documents` table keyed by full document path, with
`collection_path`, `parent_path`, `collection_id`, `data jsonb`, `create_time` and
`update_time` columns. Composite indexes become expression indexes on the
`data` paths that `firestore.indexes.json` lists. `AsyncClient` gets the same
transport through its async GAPIC stub.

Rejected adapter alternatives:

- A duck-typed fake client: the SDK class patches above would stop applying.
- A per-domain rewrite onto `database/ports/`: correct long term, but it touches
  all 361 database modules before the MVP can run.
- The Firestore emulator: not durable and not an AWS service.

The `omi_flags` project store stays unset. `store_target()` then returns `None`
and flags resolve to defaults.

### Auth: Cognito access token, `uid = sub`

**Token shape** (pool `ap-southeast-4_jA7oAYYrp`):

| Claim | Access token | ID token |
| --- | --- | --- |
| `iss` | `https://cognito-idp.ap-southeast-4.amazonaws.com/ap-southeast-4_jA7oAYYrp` | same |
| `token_use` | `access` | `id` |
| client identifier | `client_id` | `aud` |
| `sub` | user UUID | user UUID |
| `scope` | `openid email profile` | absent |
| `email`, `cognito:username` | `username` only | present |
| `alg` | RS256 | RS256 |

JWKS: `https://cognito-idp.ap-southeast-4.amazonaws.com/ap-southeast-4_jA7oAYYrp/.well-known/jwks.json`.
OAuth endpoints are on `https://auth.home.sauhsoj.wtf/oauth2/{authorize,token,revoke}`
and `/logout`. The pool supports S256 PKCE.

**Backend verification.** The client sends the access token as
`Authorization: Bearer <token>`, which `app/lib/backend/http/shared.dart`
already does for the Firebase token. `verify_token` checks, in order: RS256
signature against the cached JWKS, `iss` equals the pool issuer,
`token_use == "access"`, `client_id` in an allowlist env var
(`COGNITO_ALLOWED_CLIENT_IDS`), and `exp`. It returns `sub` as the uid. The same
function backs `_verify_ws_auth`, so `/v4/listen` gets the change for free. Map
`jwt.ExpiredSignatureError` to WebSocket close 4001 (refresh) and every other
verification failure to 1008. The existing `_get_ws_auth_close` keeps that
split for Firebase errors.

Other auth seams: `dependencies.py::get_current_user_id` (64),
`routers/oauth.py`, `utils/mcp_server/oauth.py`, plus the Firebase verify call
sites in `routers/transcribe.py`, `routers/mcp.py`, `routers/mcp_sse.py`,
`routers/apps.py`, `routers/referrals.py`, `routers/support.py` and
`routers/public_shared_conversation_chat.py`. `main.py` and `pusher/main.py`
must skip `firebase_admin.initialize_app`.

**User lookup.** Replace `database/auth.py::_firebase_get_user` with
`AdminGetUser(Username=sub)`. For sampled users in this pool, `Username` equals
`sub`. The task role needs `cognito-idp:AdminGetUser` on the pool ARN in
ap-southeast-4, which is cross-region from the ap-southeast-2 task.

**Settings that must hold:**

- `ADMIN_KEY_AUTH_ENABLED=false`, because the default `true` enables
  `<ADMIN_KEY><uid>` impersonation in `verify_token`.
- `LOCAL_DEVELOPMENT` unset. Its `'123'` uid bypass in `verify_token` fires
  when no Google credential variable is set, which will be true on AWS.
  The new verifier must drop that branch.

**Accepted gaps:**

- Offline verification does not see revoked tokens. Accepted with a 60-minute
  access-token lifetime.
- A UUID `sub` passes `utils/referral_campaign.py` `UID_PATTERN`, but
  `FIREBASE_UID_RE` in `utils/error_class_fingerprint.py` will not redact it in
  logs. This is a minor log-privacy gap to fix in the auth card.
- `utils/api_key_families.py::api_key_family` classifies any JWT as
  `FIREBASE_FAMILY`. That label is harmless.

**New app client**, added in `~/src/sauhsoj-account-portal/lib/constructs/cognito.ts`:

| Setting | Value |
| --- | --- |
| Construct / name | `OmiMobileClient` / `omi-mobile` |
| Secret | none (public client) |
| OAuth flows | authorization code with PKCE only. No implicit grant |
| Scopes | `openid`, `email`, `profile` |
| Identity providers | `COGNITO` |
| Callback URLs | `omi://auth/callback`, `omi-beta://auth/callback`, `omi-dev://auth/callback` |
| Logout URLs | `omi://auth/logout`, `omi-beta://auth/logout`, `omi-dev://auth/logout` |
| Explicit auth flows | `ALLOW_REFRESH_TOKEN_AUTH`, `ALLOW_USER_AUTH` |
| Token validity | access 60 min, ID 60 min, refresh 30 days |
| Token revocation | enabled |
| Prevent user-existence errors | enabled |
| Managed login | a managed-login branding resource for this client |
| Stack output | `OmiMobileClientId` |

Do not reuse the existing `AppClient`. It allows the implicit grant and an
admin scope. Callback schemes are registered in
`app/android/app/src/{dev,prod}/AndroidManifest.xml` and through
`AUTH_CALLBACK_SCHEME` in the iOS xcconfigs and `Info.plist`. The client seam is
`app/lib/services/auth_service.dart`: `authenticateWithProvider`,
`_exchangeCodeForOAuthCredentials`, and `refreshIdToken`/`getIdToken`. Its
existing PKCE and `app_links` flow is re-pointed at the hosted UI endpoints
above. Risk: the `omi://` scheme collides with the official Omi app if both are
installed on one device. The fork can switch to a distinct scheme by changing
the three callback values.

### STT: Amazon Transcribe streaming

The `/v4/listen` contract (`routers/transcribe.py:191`) and Transcribe:

| Contract item | Current behaviour | Transcribe streaming | Gap |
| --- | --- | --- | --- |
| Codec | client sends `opus`, `aac`, `lc3` or `pcm8`/`pcm16` | accepts PCM, FLAC, Ogg-Opus | None. `routers/listen/receiver.py:930-965` already decodes to PCM16 before the STT socket |
| Sample rate | `sample_rate` query, default 8000 | 8-48 kHz PCM | None |
| Interim results | Soniox path emits finals only | `IsPartial` flag per result | None. Forward only `IsPartial=false` |
| Speaker labels | `SPEAKER_{idx:02d}` from Soniox | `ShowSpeakerLabel` gives `spk_N` | Map `spk_N` to `SPEAKER_NN` |
| `language=multi` | Soniox auto language | `IdentifyMultipleLanguages` with `LanguageOptions` | `UNVERIFIED`: whether multi-language ID combines with speaker labels |
| Session length | Soniox rotation in `utils/stt/streaming.py` | FAQ says up to 8 h, the quotas page gives no number (`UNVERIFIED`) | Rotate streams the way Soniox rotation does |
| Custom vocabulary | per-user keywords | needs pre-created vocabulary resources | Cut for MVP |
| Owner voiceprint | `HOSTED_SPEAKER_EMBEDDING_API_URL` service | none | Cut for MVP (leave the env var unset) |

Sources: [streaming](https://docs.aws.amazon.com/transcribe/latest/dg/streaming.html),
[StartStreamTranscription](https://docs.aws.amazon.com/transcribe/latest/APIReference/API_streaming_StartStreamTranscription.html),
[diarization](https://docs.aws.amazon.com/transcribe/latest/dg/diarization.html),
[partial results](https://docs.aws.amazon.com/transcribe/latest/dg/streaming-partial-results.html).

SDK: `aws-sdk-transcribe-streaming` (developer preview, so pin the version). The
deprecated `amazon-transcribe` package was rejected for high CPU use. A raw
WebSocket with event-stream framing is the fallback if the preview SDK breaks.

Config: `STT_SERVICE_MODELS=transcribe`, `SONIOX_API_KEY` and `MODULATE_API_KEY`
unset, `STT_GPU_FLEET_OFF` unset. `validate_streaming_stt_env`
(`utils/stt/streaming.py:861`) raises when `soniox` is listed without a key, or
when `STT_GPU_FLEET_OFF` is true without one.

### LLM: Bedrock through `langchain-aws`

- **Provider.** Add a `bedrock` provider beside `OPENAI_COMPATIBLE_PROVIDERS`
  (`utils/llm/providers.py:71`) that returns `ChatBedrockConverse`. Today every
  managed feature resolves to `('gpt-6-luna', 'openai')` in
  `utils/llm/model_config.py`. Add an `aws` profile to `MODEL_QOS_PROFILES` and
  set `MODEL_QOS=aws`.
- **Models** (verified in ap-southeast-2 with `list-inference-profiles`):
  `au.anthropic.claude-sonnet-4-5-20250929-v1:0` for `chat_agent` and
  `conv_structure`, and `au.anthropic.claude-haiku-4-5-20251001-v1:0` for the
  other features. The `au.` profiles keep inference in Australian regions. The
  task role allows only `au.*` profiles and denies `global.*`.
- **Agent loop fit.** `utils/retrieval/agentic.py::_run_openai_agent_stream`
  calls `get_llm('chat_agent')` and reads `tool_call_chunks`, which
  `ChatBedrockConverse` emits. Strip OpenAI-only kwargs (for example
  `stream_options`) in the provider. The `_AnthropicClientProxy` path
  (`agentic.py` around line 963) switches to `anthropic.AsyncAnthropicBedrock`.
- **Gateway.** `OMI_LLM_GATEWAY_FEATURE_MODE` stays unset, so features call the
  model directly.

### Hosting

| Resource | Name / value |
| --- | --- |
| VPC | `omi-vpc`, `10.42.0.0/16`, 2 AZs. Avoids the existing `10.0.0.0/16` (x2) and `172.31.0.0/16` |
| Subnets | `omi-public-a/b` (tasks, ALB), `omi-isolated-a/b` (Aurora, Valkey). No NAT gateway |
| VPC endpoint | S3 gateway endpoint on the public route tables |
| ECS cluster | `omi-cluster` (Fargate, x86_64). Existing clusters `pihole-cluster` and `KeycloakStack` are untouched |
| ECS service / task | `omi-backend`. One task with containers `backend` (port 8080) and `pusher` (port 8081). Starting size 2 vCPU and 4-6 GiB (`UNVERIFIED`; measure the image's resident size first) |
| ECR | `omi/backend`, `omi/pusher` |
| ALB | `omi-alb`, internet-facing, HTTPS 443 only, idle timeout 3600 s (the client pings every 20 s; long chat streams need headroom) |
| Target group | `omi-backend-tg`, HTTP 8080, health check `GET /health` (`routers/desktop_core.py:77`) |
| ACM certificate | `omi.home.sauhsoj.wtf`, DNS-validated in zone `Z01865332JZ618N8XF2YM`, ap-southeast-2 |
| Route53 | `omi.home.sauhsoj.wtf` A/AAAA alias to `omi-alb` |
| Security groups | `omi-alb-sg` (443 from anywhere), `omi-task-sg` (8080 from `omi-alb-sg` only; 8081 not exposed), `omi-data-sg` (5432 and 6379 from `omi-task-sg`) |
| Aurora | cluster `omi-aurora`, PostgreSQL 16.3 or later, Serverless v2, min 0.5 ACU / max 4 ACU, one writer, database `omi` |
| Valkey | `omi-valkey`, `cache.t4g.micro`, one node, cluster mode disabled, TLS and AUTH on |
| Secrets Manager | see below |
| Logs | CloudWatch log groups `/ecs/omi-backend` and `/ecs/omi-pusher`, 30-day retention |
| IaC | a new CDK app under `infrastructure/aws/`. The existing `infrastructure/opentofu` targets GCP only |

Tasks run in public subnets with public IPs so they reach Bedrock, Transcribe,
Cognito and GitHub without NAT. Inbound access is limited by `omi-task-sg`.
EC2 was rejected because it adds patching for no saving at this size. Lambda
cannot hold the `/v4/listen` WebSocket.

**Pusher sidecar.** `schedule_finalization`
(`routers/listen/conversations.py:330`) logs "Pusher unavailable; finalization
remains queued" and returns `False` when no pusher client exists and
listen-finalization dispatch is off. Conversations would never finalize. The
pusher image (`backend/pusher/Dockerfile`) therefore runs in the same task on
port 8081, with `HOSTED_PUSHER_API_URL=http://localhost:8081`. It stays off the
ALB, because `/v1/trigger/listen` takes a `uid` query parameter without
authenticating it.

**S3 buckets** (names are `omi-<purpose>-717494614307-apse2`; versioning off,
SSE-S3, public access blocked):

| Env var | Bucket | MVP use |
| --- | --- | --- |
| `BUCKET_PRIVATE_CLOUD_SYNC` | `omi-private-cloud-sync-717494614307-apse2` | live audio archive (private cloud sync defaults on) |
| `BUCKET_CANONICAL_AUDIO` | `omi-canonical-audio-717494614307-apse2` | canonical audio for finalization |
| `BUCKET_AUDIO_DERIVED` | `omi-audio-derived-717494614307-apse2` | derived audio artifacts |
| `BUCKET_SPEECH_PROFILES` | `omi-speech-profiles-717494614307-apse2` | `GET /v3/speech-profile` existence check |
| `BUCKET_CHAT_FILES` | `omi-chat-files-717494614307-apse2` | chat attachments |
| `BUCKET_POSTPROCESSING` | `omi-postprocessing-717494614307-apse2` | reprocess inputs |

Other `BUCKET_*` variables (`BUCKET_BATCH_PIPELINE`, `BUCKET_SCREEN_FRAMES`,
`BUCKET_TEMPORAL_SYNC_LOCAL`, `BUCKET_MEMORIES_RECORDINGS`,
`BUCKET_DESKTOP_UPDATES`, `BUCKET_PLUGINS_LOGOS`, `BUCKET_APP_THUMBNAILS`) stay
unset for MVP.

GCS semantics in the S3 client: `if_generation_match=0` becomes
`If-None-Match: *`. Other generation preconditions become `If-Match` on the
ETag plus a metadata counter, which is a correctness risk to test. Signed URLs
become presigned URLs, and resumable uploads become multipart uploads.

**Secrets Manager entries** (prefix `omi/`):

| Secret | Consumer |
| --- | --- |
| `omi/encryption-secret` | `ENCRYPTION_SECRET` (per-uid HKDF key) |
| `omi/memory-v3-cursor-secret` | `MEMORY_V3_CURSOR_SECRET`, required by `utils/memory/universal_list_cursor.py` |
| `omi/sync-content-id-secret` | `SYNC_CONTENT_ID_SECRET` |
| `omi/valkey-auth` | `REDIS_DB_PASSWORD` |
| RDS-managed master secret for `omi-aurora` | Postgres DSN for the adapter, `CELLS_PG_DSN`, `JOBSTORE_PG_*` |
| `omi/github-token` (optional) | `GITHUB_TOKEN` for firmware release reads |

### Runtime settings

- `OMI_ENV_STAGE=dev`. `utils/env_loader.py:25` accepts only `prod`, `dev`,
  `local` and `offline`, and `utils/sync/stage.py::current_stage` raises when it
  is unset.
  - `prod` makes `validate_account_deletion_dispatch_configuration`
    (`utils/cloud_tasks.py`, called at `main.py:249`) require Cloud Tasks.
  - `local` and `offline` turn GCS writes into no-ops in `utils/other/storage.py`.
  - `dev` costs four effects: it skips Stripe price validation, prefixes sync
    Redis keys and collections with `dev`, turns on stage profiling and parity
    telemetry, and lets the LLM gateway route features.
  - This choice is `UNVERIFIED` until the first deploy boots clean.
- `MEMORY_ENABLED=on`. Memory fails closed when this is unset
  (`config/memory_rollout.py`).
- `MARKETPLACE_APP_REVIEWERS=<owner sub>` for the transcription and processing
  allowance. The chat quota does not honour reviewers, so seed the owner's
  `users/{sub}` document with `subscription.plan=unlimited`,
  `subscription.status=active` and a far-future `current_period_end`.
- `OMI_ROLE` unset, so `app_roles.py` mounts every router role.
- `DIRECTORY_ROUTING_MODE` unset (off), `FAIR_USE_ENABLED=false`,
  `STT_CAP_DISPATCH_ENFORCEMENT=false`.

### MVP cut list

| Feature | Switch that disables it | Effect |
| --- | --- | --- |
| Stripe billing | `STRIPE_API_KEY`, `STRIPE_*_PRICE_ID` unset. `validate_stripe_price_ids` skips under `dev` and skips empty price ids | Paid plans invisible. Owner is seeded `unlimited` |
| PostHog | `POSTHOG_API_KEY`, `POSTHOG_PROJECT_API_KEY`, `POSTHOG_HOST` unset on the backend. App guards on `Env.posthogApiKey != null` | No analytics |
| LangSmith | `LANGSMITH_TRACING=false`, no `LANGSMITH_API_KEY` | No LLM tracing |
| Plugins / app store | No switch exists. Routers stay mounted, and the MVP client loop never calls `/v1/apps*` or `/v2/apps*`. Seam to add: a router denylist in `app_roles.py::mount_routers` (329) | Store pages return empty or error |
| Typesense | `TYPESENSE_HOST`, `TYPESENSE_API_KEY` unset | `POST /v1/conversations/search` returns 500 because `search_conversations` raises. The app search box is unsupported in MVP |
| Pinecone | `PINECONE_API_KEY`, `PINECONE_INDEX_NAME` unset | Index is `None`. Upserts return `False`, queries return empty, and chat retrieval (`utils/retrieval/tool_services/conversations.py:330`) skips the vector branch |
| Twilio phone calls | `TWILIO_*` unset | `twilio_service` raises, so phone-call routes fail. Unused by the MVP loop |
| Telegram / iMessage messaging | `OMI_TELEGRAM_ENABLED=off`, `OMI_IMESSAGE_ENABLED=off` (defaults) | `start_adapters` returns `None` |
| Third-party integrations (Google Calendar, Gmail, Tasks) | `GOOGLE_TASKS_CLIENT_ID` and the other provider client ids unset | Integration tools report unconfigured |
| Push notifications | Client never registers an FCM token, so `_send_to_user` returns 0. Seam to add: `OMI_PUSH_PROVIDER` in `utils/notifications.py` | No push |
| Firebase sign-in (`/v1/auth/*`) | Client uses the Cognito hosted UI instead | Google and Apple buttons removed from the fork |
| Client Firebase SDKs (Messaging, Crashlytics) and Intercom | Removed from the fork's Flutter build | No crash reports or support chat |
| Offline sync upload (`/v2/sync-local-files`, `/v2/sync-capture-manifest`) | Deferred to phase 2. The WAL keeps audio locally on 503 (`app/lib/services/wals/CUSTODY.md`) | Phase 2 path is Transcribe batch from S3 |
| Custom vocabulary, owner voiceprint | `HOSTED_SPEAKER_EMBEDDING_API_URL` unset | Lower diarization quality |
| Perplexity web search tool | `PERPLEXITY_API_KEY` unset | Chat web-search tool unavailable |

## Consequences

- **Easier.** One Postgres engine serves the document adapter, CellStore and the
  jobstore. Every MVP model call stays in AWS under one IAM role. Auth needs no
  Google project.
- **Harder.** The GAPIC adapter must reproduce Firestore query semantics:
  ordering, cursors, `in` and `array-contains` filters, transforms, and
  preconditions. It needs a conformance suite run against the offline SDK
  fixture before any route is switched. Firestore's 1 MiB document limit and
  ordering rules become adapter tests, not database guarantees.
- **Watch.** Aurora auto-pause versus the reconcile loops. Transcribe
  multi-language with diarization. The cross-region `AdminGetUser` call. The
  `omi://` scheme collision. Fargate memory headroom with two containers.
- **Upstream drift.** New upstream Firestore features reach the fork through
  the adapter. A feature the adapter does not implement fails loudly at the
  transport rather than silently.

## Implemented in

Nothing yet. Implementation cards under the omi-aws initiative cite this ADR
when they change the seams named above.
