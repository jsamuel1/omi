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
| Firebase Auth (ID token verify) | Cognito access token, verified with PyJWT 2.15.1 `PyJWKClient` against the pool JWKS, with failures raised as provider-neutral types (see "Verification failures") | Cognito ID token (carries no `client_id` claim and is meant for the client, not resource servers) | `utils/other/endpoints.py::verify_token` (143), `_verify_ws_auth` (390), `dependencies.py::get_current_user_id` (64), and the other boundaries under "Every direct auth boundary"; new `utils/other/auth_tokens.py` | No |
| Firebase Admin user lookup | `cognito-idp:AdminGetUser(Username=sub)` behind a normalizing adapter that returns the Firebase `UserRecord` attribute shape (see "User lookup adapter") | `ListUsers` with a `sub` filter (needs a second permission, returns a list to page and disambiguate) | `database/auth.py::_firebase_get_user` (line 13), plus the raw `auth.get_user` sites listed in "User lookup adapter" | No |
| Firebase custom-token sign-in (`/v1/auth/*`, Google and Apple) | Cut. Cognito hosted UI with PKCE replaces it | Federating Google and Apple IdPs into the pool now (the pool has COGNITO as its only IdP, and the owner admin-creates users) | `routers/auth.py` stops being called by the client | No |
| GCS buckets | S3, one bucket per `BUCKET_*` env var, behind a new `S3StorageClient` | One shared bucket with prefixes (breaks per-bucket lifecycle and IAM scoping that the env-var split already implies) | `utils/other/local_storage.py::create_storage_client` (line 348), plus direct `google.cloud.storage` importers listed below | No |
| Cloud Tasks (durable listen-finalization dispatch) | A new `LISTEN_FINALIZATION_DISPATCH_MODE=in_process` mode: a bounded asyncio dispatcher in the backend container runs the existing worker handler. The durable job rows and the existing startup and periodic reconcilers are the recovery owner (see "Durable finalization") | SQS plus an in-task consumer (extra queue, DLQ and IAM, but the consumer dies with the same task, so it protects nothing the durable rows do not). Inline pusher dispatch (disables every durable recovery path). Step Functions (a workflow engine for a single wake message). Lambda workers (15-minute limit is below the 1500 s dispatch deadline plus retries) | `utils/cloud_tasks.py` `is_listen_finalization_dispatch_enabled` (328), `is_listen_finalization_dispatch_configured` (341), `is_listen_finalization_tasks_configured` (346), `enqueue_listen_finalization_job` (649); `routers/conversation_finalization.py::run_listen_finalization_job` (106) | No |
| Memorystore / Redis Cloud | ElastiCache for Valkey, node-based, `cache.t4g.micro`, cluster mode disabled | ElastiCache Serverless (cluster-mode only; the code uses 19 `r.eval`, 12 `register_script`, 6 pipelines and 2 `mget`, which risk CROSSSLOT) | `database/redis_connection.py` gains `redis_tls_kwargs()` and a `REDIS_DB_TLS` switch, and every production constructor routes through it (see "Valkey: TLS on every client") | No |
| Soniox / Modulate streaming STT | Amazon Transcribe streaming over a raw SigV4 WebSocket on Python 3.11, as a new `transcribe` provider and `TranscribeStreamingSocket(STTSocket)` | Self-hosted Parakeet on GPU (GPU cost floor for one user) | The closed STT policy seams listed under "STT: seam contract" | No |
| OpenAI / OpenRouter / Anthropic LLMs | Amazon Bedrock through `langchain-aws` `ChatBedrockConverse`, `au.` inference profiles | Bedrock's OpenAI-compatible endpoint (serves gpt-oss only, no Claude) | `utils/llm/providers.py` (new `get_or_create_bedrock_llm`), `utils/llm/clients.py::get_llm` (762, explicit `bedrock` dispatch before the managed gateway branch at 915), `utils/llm/request_options.py` (new, bind-time options), `utils/llm/model_config.py` (new `aws` `MODEL_QOS` profile). Direct gateway callsites are ported or cut in "LLM: Bedrock through `langchain-aws`" | No |
| OpenAI embeddings | Bedrock `amazon.titan-embed-text-v2:0` (1024-dim) via `BedrockEmbeddings`, behind a Titan input fitter that retries on Titan's own over-token rejection. No embedding call is made while the vector index is cut (see "Embeddings: Titan input bound and the cut index") | Cohere Embed on Bedrock (no gain at this scale, one more model to approve) | `utils/llm/clients.py::_OpenAIEmbeddingsProxy` (356) and `generate_embedding` (1076); `utils/conversations/process_conversation.py::save_structured_vector` (2175) | No |
| OpenAI Files (chat attachments) | S3 `BUCKET_CHAT_FILES` for the bytes, sent to Bedrock Converse as image and document content blocks, with upload admission narrowed to what Converse accepts (see "Attach contract") | Keeping OpenAI Files (not an AWS service, and a second LLM vendor for one feature) | `utils/other/chat_file.py` `FileChatTool.upload` (262), `upload_video`, `_ask_files_stream` (397), `_ask_files` (462), `download_owned_chat_file` (115); `models/chat.py` `chat_file_is_document` | No |
| Gemini / ElevenLabs TTS (`/v2/tts/synthesize`) | Cut for MVP. Amazon Polly later | Polly now (the client already falls back to the system voice) | `routers/tts.py` | No |
| Pinecone (conversation vectors) | Cut for MVP. pgvector on the same Aurora cluster later | OpenSearch Serverless (OCU minimum is a standing cost for one user) | `database/vector_db.py` (`PINECONE_API_KEY`/`PINECONE_INDEX_NAME` at 144-155) | No |
| Typesense (conversation search) | Cut for MVP. Postgres full-text search later | OpenSearch managed domain (standing instance cost) | `utils/conversations/search.py::search_conversations` (342) | No |
| Google Maps Static API (`GET /v1/static-map`, `utils/static_map.py:343`, `GOOGLE_MAPS_API_KEY`) | Cut for MVP via `OMI_DISABLED_ROUTERS=static_map`. Amazon Location Service static maps later | Keeping the Google key (a GCP credential in an AWS-only port) | `routers/static_map.py`, `utils/static_map.py::fetch_static_map` | No |
| Firebase Cloud Messaging push | Cut for MVP via `OMI_DISABLED_ROUTES` (see "Cut switches"), with the app's notification service made local-only (see "Client: Firebase retirement"). SNS mobile push to APNs later | Amazon Pinpoint (end of support 2026-10-30) | `POST /v1/users/fcm-token` (`routers/notifications.py:35`), `utils/notifications.py`; client `services/notifications/notification_service.dart` | No |
| GitHub Releases (firmware manifest) | Kept as is. Not GCP | S3 mirror of firmware binaries (duplicates a public source that already has a Redis LKG cache) | `utils/github_releases.py` | n/a |
| Secret Manager / env files | AWS Secrets Manager, injected as ECS task secrets | SSM Parameter Store SecureString (no managed rotation for the RDS secret) | ECS task definition `secrets` | n/a |
| GKE / Cloud Run hosting | ECS Fargate behind an ALB | App Runner (no WebSocket support) | Container image from `backend/Dockerfile` and `backend/pusher/Dockerfile` | n/a |
| Base image `gcr.io/based-hardware-dev/python:3.11-slim-forky` (`backend/Dockerfile:1`, `backend/pusher/Dockerfile:1`) | `public.ecr.aws/docker/library/python:3.11-slim`, passed as the existing `PYTHON_BASE_IMAGE` build arg | Mirroring the GCR image into ECR (keeps a private upstream image the fork cannot rebuild) | `PYTHON_BASE_IMAGE` build arg | n/a |
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
  The backend runs the durable-finalization reconcilers every 60 s
  (`main.py:278`, `_periodic_listen_finalization_reconcile`, 60 s with quiet
  finalization on), and the port depends on them for recovery (see "Durable
  finalization"). Each tick reads Aurora, so the cluster never reaches the
  300 s minimum auto-pause idle period. Auto-pause is therefore not a cost
  lever for this design: run at a 0.5 ACU minimum and measure. Pricing:
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

**Error types.** The adapter raises `google.api_core.exceptions` types
(`NotFound`, `AlreadyExists`, `FailedPrecondition`, `Aborted`,
`DeadlineExceeded`, `ServiceUnavailable`), never `psycopg` errors. Nine modules
catch `GoogleAPICallError` and map it to a retry or a 503:
`database/action_items.py`, `database/batch_pipeline_reads.py`,
`database/batch_privacy_write_fence.py`, `database/chat_first_delivery_attempts.py`,
`routers/conversation_tasks.py` (the MVP task-presence route),
`utils/batch_pipeline/merge.py`, `utils/conversations/processing_retry.py`,
`utils/sync/failure_reason.py` and `utils/sync/pipeline.py`. A Postgres error
that escaped as its own type would turn those retries into 500s. The adapter
maps serialization failures to `Aborted` and statement timeouts to
`DeadlineExceeded`.

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
function backs `_verify_ws_auth`, so `/v4/listen` gets the change for free.
`main.py` and `pusher/main.py` must skip `firebase_admin.initialize_app`.

**Verification failures.** Today's callers catch Firebase exception classes:
`InvalidIdTokenError` at `utils/other/endpoints.py:227`, `:292` and `:360`,
`(InvalidIdTokenError, CertificateFetchError)` in `_verify_ws_auth` (407), and
`InvalidIdTokenError` in `routers/transcribe.py:281`. PyJWT raises its own
family, so a bare swap would turn every rejected token on those HTTP routes
into a 500. The verifier therefore never lets a PyJWT or `PyJWKClient`
exception escape. It raises one of three provider-neutral types from a new
`utils/other/auth_tokens.py`, and two mapping functions in the same module
turn them into the transport answer:

| Type | Meaning | Raised for (PyJWT 2.15.1) |
| --- | --- | --- |
| `AuthCredentialExpired(AuthCredentialError)` | The client sent a well-formed token past `exp`. It should refresh | `jwt.ExpiredSignatureError` |
| `AuthCredentialInvalid(AuthCredentialError)` | The client sent a token this service will never accept | every other `jwt.InvalidTokenError` subclass (`DecodeError`, `InvalidSignatureError`, `InvalidIssuerError`, `InvalidAudienceError`, `InvalidAlgorithmError`, `ImmatureSignatureError`, `InvalidIssuedAtError`, `MissingRequiredClaimError`), a header `alg` other than RS256, `token_use != "access"`, a `client_id` outside the allowlist, and a `kid` that is still unknown after a JWKS refresh |
| `AuthProviderUnavailable` (not an `AuthCredentialError`) | The server could not evaluate the token | `jwt.PyJWKClientError` (including `PyJWKClientConnectionError`) and `jwt.PyJWKSetError` from fetching the key set, and an unknown `kid` while a refresh is suppressed by the cooldown below |

The split between "unknown kid" and "key set unavailable" cannot come from
`PyJWKClient.get_signing_key_from_jwt`: it raises the same plain
`PyJWKClientError` for a `kid` with no match and for a JWKS endpoint that
returned a non-object. The verifier does the lookup itself instead:
`jwt.get_unverified_header` (a `DecodeError` is invalid), then
`client.get_signing_keys()` inside a `try` that maps every `PyJWTError` to
`AuthProviderUnavailable`, then the static `PyJWKClient.match_kid`. On no
match it calls `get_signing_keys(refresh=True)` at most once per 30 s (a
module-level monotonic timestamp under a lock). A `kid` still unknown after a
successful refresh is `AuthCredentialInvalid`. A `kid` unknown during the
cooldown is `AuthProviderUnavailable`, because the server has not checked the
current key set. The client is built with `timeout=5` and `lifespan=300`, so a
hung JWKS endpoint holds a worker for 5 s, not PyJWKClient's default 30 s.

Any other exception from the verifier is a bug and propagates. HTTP callers
answer 500 for it, and WebSocket callers close with 1011.

`http_auth_exception(error)` and `ws_auth_close(error)` are the only mapping:

| Failure | HTTP | WebSocket close | Log level |
| --- | --- | --- | --- |
| `AuthCredentialInvalid` | 401 `Invalid authorization token` | 1008 | warning |
| `AuthCredentialExpired` | 401 `Invalid authorization token` | 4001 | warning |
| `AuthProviderUnavailable` | 503 `{"code": "auth_provider_unavailable", "retryable": true}`, `Retry-After: 1` | 1013 | error |
| `ExecutorSaturatedError` around the verify call | 503, `Retry-After: 1` (`dependencies.py:73-78`, `endpoints.py::_executor_saturated_http_exception`, 70) | 1013 (`run_critical_ws`, 79) | unchanged |
| Unexpected exception | 500 | 1011 | error |

The client already treats these correctly. On HTTP, only a 401 triggers
refresh and replay (`app/lib/backend/http/shared.dart:380`), so a 503 never
signs the user out. On the socket, 4001 refreshes the token and reconnects
(`services/capture/capture_controller.dart:3470`), 1008 is terminal
(`services/capture/capture_live_recovery.dart:57`), and 1013 and 1011 are
not in that terminal set, so live recovery retries them with backoff. Today
`_get_ws_auth_close` (446) maps Google's `CertificateFetchError` to 4001,
which makes the client refresh a token that was never the problem. It also
sniffs `'expired'` and `'certificate'` in exception text. Both go: the close
code comes from the type alone. Close 4004 (relogin) keeps its constant but
has no auth producer, because offline verification cannot see revocation
(accepted gap below).

**Every direct auth boundary.** Each site below calls the verifier and maps
its failures through the two functions, so no caller names a provider
exception:

| Boundary | Callers | Port |
| --- | --- | --- |
| `utils/other/endpoints.py::get_current_user_uid` (271), `get_current_user_uid_no_byok_validation` (334), `get_current_user_uid_for_deletion_status` (199), and `get_current_user_uid_readonly` (257) through the last | every authenticated MVP HTTP row (census, "Shared auth chain") | `except AuthCredentialError, AuthProviderUnavailable` → `raise http_auth_exception(e)`, replacing the three `except InvalidIdTokenError` clauses |
| `dependencies.py::get_current_user_id` (64) | `routers/api_key_management.py`, `routers/mcp.py` (not reached by the app) | `run_blocking(critical_executor, verify_token, ...)`. Saturation keeps its 503. The verifier's types go through `http_auth_exception`. The bare `except Exception` → 401 (79-81) goes, so an unavailable key set is a 503 |
| `_verify_ws_auth` (390), used by `get_current_user_uid_ws_listen` (462) and `get_current_user_uid_ws` (513) | `WS /v4/listen` | `except (AuthCredentialError, AuthProviderUnavailable)` → `WebSocketException(ws_auth_close(e))`. The `except Exception` → 1008 (412) becomes 1011 |
| `get_current_user_uid_from_ws_message` (583), with `_verify_user_uid_from_ws_message` (545) | `WS /v4/web/listen` (`routers/transcribe.py:246`, the web app, not the Flutter app) | Translates the verifier's types into `WebSocketException` the same way. `routers/transcribe.py` drops its `except InvalidIdTokenError` clause (281). Its existing `except WebSocketException` (285) then sends `auth_response: false` and closes with the mapped code. Format errors stay `ValueError` → 1008 |
| `routers/firmware.py::_verified_uid` (347) | `GET /v2/firmware/latest` | Fail-soft as today: any failure returns no uid and the release allowlist does not match. It logs `AuthProviderUnavailable` at warning, so a JWKS outage is visible |
| `utils/support_auth.py::get_support_caller_uid` (11) | `/v1/support/*` (not reached by the app) | Calls `verify_token` instead of `auth.verify_id_token`, then `http_auth_exception` (401 or 503) |
| `routers/public_shared_conversation_chat.py` (100-114) | `POST /v1/conversations/shared/chat` with `X-Omi-User-Id-Token` (the web share page, not the Flutter app) | `run_blocking(critical_executor, verify_token, ...)`. A credential error keeps the 401 `invalid_token` body. `AuthProviderUnavailable` and saturation answer 503 `auth_provider_unavailable` instead of the bare `except Exception` → 401. The router is cut in the MVP (`OMI_DISABLED_ROUTERS=public_shared_conversation_chat`, see "Cut switches"), so this mapping applies only when it is re-enabled |
| `routers/oauth.py::oauth_token` (165) and `utils/mcp_server/oauth.py::mcp_authorize_consent` (307) | browser pages that sign in with the Firebase JS SDK (`templates/oauth_authenticate.html`, `templates/mcp_oauth_authorize.html:9-12`) and post a `firebase_id_token` | Cut: `OMI_DISABLED_ROUTERS=oauth` and `OMI_DISABLED_ROUTES=GET /authorize;POST /authorize` (see "Cut switches"). Without Firebase those pages cannot produce a token. A Cognito hosted-UI consent page is follow-up work |
| `routers/apps.py:1864` (`POST /v1/apps/migrate-owner`) | anonymous-account migration | Cut with the `apps` router. The pool has no anonymous users |
| `routers/auth.py` | Firebase sign-in | Cut with the `auth` router |

**Tests (auth card).**

- New `tests/unit/test_auth_tokens.py`, with an RSA key pair and a stubbed
  JWKS endpoint. A valid access token returns `sub`. Each PyJWT failure in the
  type table raises its mapped type: expired, bad signature, wrong `iss`, any
  `aud` (a Cognito ID token), `alg` HS256 or none, a future `nbf`, a missing
  `client_id`, `token_use: id`, a disallowed `client_id`, and garbage input.
  For the key set: a connection error, a non-object body and an empty set each
  raise `AuthProviderUnavailable`. An unknown `kid` raises
  `AuthCredentialInvalid` after one refresh, and `AuthProviderUnavailable`
  when a second unknown `kid` arrives inside the cooldown. No PyJWT exception
  type escapes any case.
- New `tests/unit/test_auth_transport_mapping.py`, parametrized over every
  boundary in the table above × {invalid, expired, unavailable, saturated}. It
  asserts the status, detail and `Retry-After` for HTTP, and the close code
  (plus `auth_response: false` for `/v4/web/listen`) for WebSocket. The
  firmware case asserts the request still succeeds without the allowlisted
  release.
- `tests/unit/test_ws_auth_handshake.py`: the certificate-fetch case expects
  1013, not 4001. The revoked → 4004 case and the two message-sniffing cases
  (209, 228) are deleted. The unexpected-error case (266) expects 1011.
- `tests/unit/test_ws_auth_rejection_logging.py`: credential types log at
  warning and `AuthProviderUnavailable` at error.
  `docs/operational/ws-auth-rejection-severity.md` changes its table to the
  three types.
- `tests/unit/test_dependency_async_boundaries.py`: `get_current_user_id`
  still offloads to `critical_executor`, and an unavailable key set now gives
  503 (the case at 476 gives 401 for an invalid token).
- `tests/unit/test_verify_token_admin_and_local_dev_gating.py` keeps the
  `ADMIN_KEY` gating cases and loses the `'123'` local-development cases.
- 30 test modules stub `firebase_admin.auth` exception classes into
  `sys.modules` because `endpoints.py` imports them
  (`grep -rl "InvalidIdTokenError\|CertificateFetchError" backend/tests`).
  They drop those stubs. A guard test fails if `utils/other/endpoints.py`,
  `dependencies.py`, `utils/support_auth.py`, `routers/transcribe.py` or
  `routers/public_shared_conversation_chat.py` imports `firebase_admin`.

**User lookup adapter.** `database/auth.py::get_user_from_uid` keeps its
contract: it returns `None` or a dict with exactly the keys `uid`, `email`,
`email_verified`, `phone_number`, `display_name`, `photo_url` and `disabled`,
and it never raises. `get_user_name`, `get_user_full_name`,
`meeting_context_pack.resolve_owner_identity` and the email and notification
callers read those keys, so none of them change. Only the SDK boundary changes.
`_firebase_get_user` becomes `_identity_get_user(uid) -> IdentityRecord | None`,
which calls `AdminGetUser(UserPoolId=COGNITO_USER_POOL_ID, Username=uid)`
through a cached `cognito-idp` client in `COGNITO_REGION` (ap-southeast-4).

`AdminGetUser` returns a mapping, not an object. `Username`, `Enabled` and
`UserCreateDate` are top-level keys, and every user value is a string inside
`UserAttributes` as `{Name, Value}` pairs. The adapter folds that list into a
dict once and builds a frozen `IdentityRecord` dataclass:

| `IdentityRecord` field | Source | Conversion and default |
| --- | --- | --- |
| `uid` | attribute `sub` | must equal the requested uid (invariant below) |
| `email` | attribute `email` | string, `None` when absent |
| `email_verified` | attribute `email_verified` | `True` only for the string `"true"`; absent or anything else is `False` |
| `phone_number` | attribute `phone_number` | string, `None` when absent |
| `display_name` | attribute `name` | string, `None` when absent or empty. The pool schema has `name`; 7 of its 9 users set it |
| `photo_url` | attribute `picture` | string, `None` when absent. Firebase's `photo_url` has no Cognito field of that name; `picture` is the OIDC standard attribute the pool schema carries |
| `disabled` | top-level `Enabled` | `not Enabled`; a missing key reads as enabled |
| `creation_timestamp_ms` | top-level `UserCreateDate` | epoch milliseconds, replacing Firebase `user_metadata.creation_timestamp` |

`get_user_from_uid` builds its dict from the first seven fields, so the dict
keys and value types match what Firebase produced.

**Lookup key invariant.** The adapter passes the uid as `Username` and accepts
the response only when its `sub` attribute equals the requested uid exactly.
On a mismatch it logs `identity_lookup outcome=key_mismatch` and returns
`None`. The invariant holds because the pool's `UsernameAttributes` is
`["email"]` (CDK `signInAliases: { email: true }` in
`sauhsoj-account-portal/lib/constructs/cognito.ts:55`, immutable after pool
creation). In such a pool Cognito generates each username as the user's `sub`,
and `AdminGetUser` also accepts the email alias as `Username`. On 2026-11-24,
`list-users` showed `Username == sub` for all 9 users. The check closes the
alias path: a uid that happens to be an email address would otherwise resolve
to that user's record. A deploy check runs `describe-user-pool` and fails
unless `UsernameAttributes == ["email"]` and `AliasAttributes` is empty.

**Errors.** `get_user_from_uid` maps each outcome to its current behaviour:

| `AdminGetUser` outcome | Adapter | `get_user_from_uid` |
| --- | --- | --- |
| `UserNotFoundException` | returns `None` | `None`, as Firebase `UserNotFoundError` gives today |
| `sub` mismatch | returns `None` and logs `key_mismatch` | `None` |
| `NotAuthorizedException`, `AccessDeniedException`, `ResourceNotFoundException` | raises `IdentityLookupUnavailable` | logs `identity_lookup outcome=unavailable code=<code>` at error level, calls `record_fallback(component='other', reason='other', outcome='degraded')`, returns `None` |
| `TooManyRequestsException`, timeouts | botocore `standard` retry mode, 3 attempts, 2 s connect and read timeouts, then as above | `None` |
| empty uid | no call | `None` |

A misconfigured IAM policy or pool id therefore shows up as a fallback
counter instead of silently turning every owner into "The User".

**Other raw Firebase user reads** move to the same adapter, so no caller reads
a Firebase `UserRecord`:

| Site | Today | Port |
| --- | --- | --- |
| `utils/notifications.py:32` `_get_user` | `auth.get_user`, then `.display_name` and `.email` (559-561) | `_identity_get_user`, same attribute names |
| `routers/mcp.py:335` `_get_user_contact` | `.display_name`, `.email`, `.phone_number` | `_identity_get_user` |
| `routers/referrals.py:99` | `user_metadata.creation_timestamp` | `IdentityRecord.creation_timestamp_ms`. The referral routes are deferred but served (census "Partition appendix") |
| `routers/support.py:32` `resolve_support_target` | `get_user_by_email(email).uid` | `AdminGetUser(Username=email)` returns `sub`, since email is the username attribute. `UserNotFoundException` → 404 |
| `services/users/account_deletion.py:487`, `:631` | `get_user`, `revoke_refresh_tokens` | Existence check uses `_identity_get_user`. Revocation does nothing in the shared pool: `AdminUserGlobalSignOut` would also sign the user out of every other `sauhsoj.wtf` app. The account-deletion fence (census, "Shared auth chain") already refuses the user's remaining access tokens |
| `utils/other/endpoints.py:816` `delete_account` | `auth.delete_user` | Deletes no Cognito user. The pool is shared, so omi deletes only its own data |
| `utils/other/endpoints.py:139` `get_user` | `auth.get_user` | Deleted. No production code calls it; the seven unit tests that assign `endpoints.get_user = MagicMock()` drop that line |

The task role gets `cognito-idp:AdminGetUser` on the pool ARN in
ap-southeast-4, which is cross-region from the ap-southeast-2 task, and
nothing else in that pool. `AdminDeleteUser`, `AdminUserGlobalSignOut` and
`ListUsers` are not granted.

**Rejected: `ListUsers` with a `sub` filter.** It needs `cognito-idp:ListUsers`,
which lets the task enumerate every user of the shared pool, and it returns a
list to page and disambiguate. With `Username == sub` enforced, `AdminGetUser`
is one exact read.

**Tests (auth card).** With a botocore `Stubber` on the `cognito-idp` client:

- A full `AdminGetUser` response yields the seven-key dict with Python types:
  `bool` for `email_verified` and `disabled`, `str` or `None` for the rest.
- `email_verified` of `"false"`, `"FALSE"` and absent each read as `False`.
  `Enabled: False` gives `disabled: True`. Absent `name`, `picture` and
  `phone_number` give `None`.
- A response whose `sub` differs from the requested uid returns `None`.
- `UserNotFoundException` returns `None` with no fallback. `AccessDeniedException`
  returns `None`, logs `unavailable` and records one fallback.
- `get_user_name` returns the first word of `name`. With no `name` it falls
  through to the Firestore profile, as `tests/unit/test_user_name_resolution.py`
  pins today.
- The tests that patch `_firebase_get_user` (`test_user_name_resolution.py`,
  `test_daily_summary_selection_mode.py`, `test_transcript_file_import.py`)
  and `notifications._get_user` (`test_notification_async_boundaries.py`)
  patch `_identity_get_user` instead and keep their assertions.
- Deployed: `GET /v2/messages` after a chat turn shows the owner's `name` in
  the system prompt's user name (`utils/llm/chat.py:525`).

**Settings that must hold:**

- `ADMIN_KEY_AUTH_ENABLED=false`, because the default `true` enables
  `<ADMIN_KEY><uid>` impersonation in `verify_token`.
- `LOCAL_DEVELOPMENT` unset. Its `'123'` uid bypass in `verify_token` fires
  when no Google credential variable is set, which will be true on AWS.
  The new verifier must drop that branch.
- `OMI_LOCAL_AUTH_SIGNING_KEY_PATH` unset. Under `OMI_ENV_STAGE=dev` the
  backend refuses to start with it set ("Client: non-production auth paths").

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
`AUTH_CALLBACK_SCHEME` in the iOS xcconfigs and `Info.plist`. Risk: the `omi://`
scheme collides with the official Omi app if both are installed on one device.
The fork can switch to a distinct scheme by changing the three callback values.

### Auth: client session migration

FirebaseAuth is a process-wide identity boundary in the app, not a detail of
`auth_service.dart`. Startup restore, account switching, outbox ownership,
background work, token refresh and sign-out all read `FirebaseAuth.instance`
directly. Repointing only the sign-in call would leave those bound to Firebase,
and the sign-in methods return Firebase `UserCredential`, which cannot carry a
Cognito session.

**Decision.** Extend the existing provider-neutral seam instead of adding a
parallel one. `AuthTokenGateway` (`services/auth/auth_token_result.dart:80`)
already exposes `currentUser` as an `AuthUserSnapshot` (60), `forceRefresh` as
a `RefreshedAuthToken` (73), and `signOut`. Its production implementation is
`_FirebaseAuthTokenGateway` (`services/auth_service.dart:27`), and 25 test and
harness files implement it as fakes (`git grep -l "implements AuthTokenGateway"`).
`AuthTokenGateway` keeps those three members unchanged, so the fakes keep
compiling. A new `AuthSessionGateway extends AuthTokenGateway` adds the session
members below, and `AuthService` holds an `AuthSessionGateway`. See
"Token-only injection seams" below for how the existing fakes reach that
field. The members:

| Member | Contract |
| --- | --- |
| `AuthUserSnapshot? get currentUser` | Synchronous, from the in-memory session. `uid` is the Cognito `sub` |
| `Stream<AuthUserSnapshot?> userChanges()` | The owner-transition stream. Each subscription first receives the current owner (`currentUser`, possibly `null`), then every transition: `restore()` completing, sign-in, sign-out, account switch, and a terminal refresh failure (`null`). Like Firebase's `authStateChanges()`, it replays on listen, so a listener attached after a cold restore still sees the owner. Replaces `authStateChanges()` and the UID-distinct `idTokenChanges()` listener (see "Auth listener migration") |
| `Stream<AuthTokenRefreshed> tokenChanges()` | Refresh-only, not replayed. Emits after each successful `forceRefresh()` or background refresh, with `ownerUid` (the `sub` the refresh token belongs to) and the new `RefreshedAuthToken`. A grant that completes after the session it started from was signed out or replaced is discarded inside the gateway: no store write and no emission. Siri's token mirror is its only consumer |
| `Future<RefreshedAuthToken?> forceRefresh()` | Refresh-token grant against `/oauth2/token`. A failed grant throws `AuthRefreshException(code, terminal:)`: token-endpoint error `invalid_grant`, `invalid_client` or `unauthorized_client` is terminal, and HTTP 429, 5xx, a timeout, a socket error or an unparseable body is transient (see "Refresh failures" in "Client: non-production auth paths") |
| `Future<void> restore()` | Reads the stored session from `flutter_secure_storage`. No network |
| `Future<AuthSignInResult> signIn()` | Hosted UI authorization code with PKCE. `AuthSignInResult` holds the snapshot and an `isNewUser` flag, replacing `UserCredential` |
| `Future<void> signOut()` | `/oauth2/revoke` on the refresh token, local wipe, then hosted-UI `/logout` |
| `Future<void> updateDisplayName(String)` | Local profile field only. The pool is admin-managed |

**Token-only injection seams.** Widening the held field also changes what the
seams that receive a gateway must supply. Two seams take a gateway from
outside `auth_service.dart`, and both keep accepting `AuthTokenGateway`:

| Seam | Callers today | Port |
| --- | --- | --- |
| `AuthService.forTesting({required AuthTokenGateway tokenGateway, ...})` (`auth_service.dart:106-125`) | 15 test and harness files (`git grep -l "AuthService.forTesting(" app/test app/integration_test`), every one passing a token-only fake | Parameter type unchanged. The initializer assigns `_gateway = asAuthSessionGateway(tokenGateway)` |
| `installLocalHarnessTokenGateway(AuthTokenGateway)` (`auth_service.dart:136`) | 11 test and harness files (`git grep -l "installLocalHarnessTokenGateway(" app/test app/integration_test`) | Signature and both assertions unchanged. Assigns `_instance._gateway = asAuthSessionGateway(gateway)` |

`asAuthSessionGateway` lives in `services/auth/token_only_session_adapter.dart`
and is the only conversion. It returns the argument when it already is an
`AuthSessionGateway`, so a session fake passes through, and otherwise wraps it
in the private `_TokenOnlySessionAdapter`:

- `currentUser`, `forceRefresh()` and `signOut()` delegate to the wrapped
  gateway.
- `userChanges()` emits `currentUser` on each listen, then nothing.
  `tokenChanges()` never emits.
- `restore()` does nothing.
- `signIn()` and `updateDisplayName()` throw `UnsupportedError`. No token-only
  caller reaches them today.

No seam casts a gateway, and no caller of either seam changes. The production
constructor `AuthService._internal()` assigns the selected session gateway
directly. `AuthService` stores one gateway field (`_tokenGateway` is renamed
`_gateway`), so every read in `auth_service.dart` goes through the session
type. `forTesting` drops `firebaseAuth` and `authorizeProviderLink` with the
link paths; their only caller, `test/unit/auth_provider_link_session_test.dart`,
is deleted with them ("Client: Firebase retirement").

The direct consumers below never hold a gateway. `AuthService` gains
`currentUser`, `userChanges()`, `tokenChanges()` and `restoreSession()`, each
delegating to the held gateway, and consumers call them on
`AuthService.instance`. A test that injects a token-only fake through either
seam therefore sees the adapter's behaviour on those paths.

**Compile coverage.** `test/unit/auth_service_token_only_seam_test.dart`
declares a fake that implements only `AuthTokenGateway`, passes it to
`AuthService.forTesting(tokenGateway: fake)` with no cast, and asserts that
`refreshIdToken()` reaches the fake's `forceRefresh`, that `userChanges()`
emits the fake's `currentUser` once, and that `restoreSession()` completes. A
second case passes an `AuthSessionGateway` fake and asserts its own
`userChanges()` stream is the one observed, so the pass-through is not
wrapped. `flutter analyze` and this test fail if either seam's parameter
narrows to `AuthSessionGateway`.

`CognitoAuthSessionGateway` implements it. It stores the access, ID and refresh
tokens in `flutter_secure_storage` (already pinned at `app/pubspec.yaml:79`) and
restores them before `runApp`. `isAnonymous` is always false. The anonymous
migration and link paths (`auth_provider.dart` lines 30-48, `auth_service.dart`
1166-1199) are deleted, because the pool has no anonymous users.

**Direct consumers to move onto the gateway** (paths relative to `app/lib/`):

| File | Lines | Use |
| --- | --- | --- |
| `main.dart` | 306 | Auth emulator setup. Deleted |
| `main.dart` | 351, 370, 383 | Restore, bootstrap and identity checks on startup |
| `main.dart` | 412, 418 | Email and uid for crash and analytics identity |
| `main.dart` | 634 | Resume-time owner check |
| `services/auth_service.dart` | 30, 42, 48, 176 | The current gateway implementation |
| `services/auth_service.dart` | 247, 284 | Google and Apple sign-in. Deleted |
| `services/auth_service.dart` | 648 `authenticateWithProvider`, 842 `_signInWithOAuthCredentials` | Becomes `signIn()` against `/oauth2/authorize` |
| `services/auth_service.dart` | 791 `signInWithLocalDevToken` | Becomes `signInLocalDev({uid})` on the local issuer gateway ("Client: non-production auth paths"). Its callers are `providers/auth_provider.dart:210` `onLocalDevSignIn` and physical qualification at `main.dart:357` |
| `services/auth_service.dart` | 156 `_terminalTokenErrorCodes`, 578 `on FirebaseAuthException` | Becomes `on AuthRefreshException`. The gateway decides terminal or transient, so the Firebase code set is deleted |
| `services/auth_service.dart` | 405-480 `getIdToken` → `_scheduleLocalDevRecovery` | The re-mint recovery is deleted ("Client: non-production auth paths") |
| `services/auth_service.dart` | 106 `AuthService.forTesting`, 136 `installLocalHarnessTokenGateway` | Both keep their `AuthTokenGateway` parameter and wrap it with `asAuthSessionGateway` ("Token-only injection seams") |
| `pages/onboarding/auth.dart` | 54, 89, 122-130 | Sign-in buttons. The Google and Apple buttons become one hosted-UI button, and the local-development button keys off the selected gateway instead of the profile |
| `services/auth_service.dart` | 513-580 `refreshIdToken` | Becomes `forceRefresh()` |
| `services/auth_service.dart` | 993, 1024 | Display-name update. Becomes `updateDisplayName` |
| `providers/auth_provider.dart` | 50, 80, 99, 187, 190 | Signed-in and owner checks |
| `providers/auth_provider.dart` | 116 `authStateChanges`, 141 and 144 `idTokenChanges` | 116 and 144 move onto `userChanges()`. The Siri-only listener at 141 is split between `userChanges()` and `tokenChanges()`. See "Auth listener migration" |
| `providers/auth_provider.dart` | 236, 268, 311-343 | `UserCredential` sign-in flow. Becomes `AuthSignInResult` |
| `services/onboarding_sync_runtime.dart` | 34 | Auth listener that wakes onboarding sync. Moves to `AuthService.instance.userChanges()`, whose replay keeps the wake on listen that `authStateChanges()` gives today |
| `services/on_device/on_device_background.dart` | 35, 49 | Owner check and auth listener for background work. 49 moves to `userChanges()`; its subscription type becomes `StreamSubscription<AuthUserSnapshot?>` |
| `services/siri_integration.dart` | 1051 | Owner check before opening a Siri result |
| `services/telemetry/client_telemetry_transport.dart` | 18, 33 | Telemetry owner id and token |
| `pages/chat/widgets/content_blocks/edit_receipt_block.dart` | 253, 258 | Owner check for receipt undo |
| `pages/home/home_deep_links.dart` | 205, 208, 256, 260, 291, 293 | Owner checks on deep-link resolution |
| `pages/home/item_list_sheet.dart` | 60, 66 | Owner check |
| `pages/onboarding/wrapper.dart` | 627-629 | Email, name and uid for onboarding |
| `pages/onboarding/variants/yinka/yinka_onboarding_flow.dart` | 156-158 | Same |
| `pages/settings/change_name_widget.dart` | 20 | Firebase `User` field |
| `utils/auth/sign_in_failure.dart` | 32 | `FirebaseAuthException` mapping. Maps OAuth error codes instead |

**Auth listener migration.** Firebase's `idTokenChanges()` is not only a
refresh stream. It is a `Stream<User?>` that replays the current user on listen
and emits on sign-in, sign-out and every refresh. `AuthProvider` relies on all
three (`providers/auth_provider.dart:116-167`), so its three listeners do not
map one-to-one onto a refresh-only stream:

| Listener today | What it depends on | Port |
| --- | --- | --- |
| 116 `authStateChanges().distinct(uid)` | Replay on listen, sign-in, sign-out (`null`) | `AuthService.instance.userChanges().distinct((p, n) => p?.uid == n?.uid)`. Body unchanged apart from the type: `User?` becomes `AuthUserSnapshot?`, and the `isAnonymous` checks read the snapshot field, which is always false |
| 144 `idTokenChanges().distinct(uid)`, the owner and token reset | Replay on listen, so a cold restore runs the token check once. A `null` event on sign-out or revocation, which clears `SharedPreferencesUtil().authToken`, `tokenExpirationTime` and `authToken`. The re-authentication path (`_requiresReauthentication`, empty or expired cached token → `getIdToken()`) on each owner change | `userChanges().distinct((p, n) => p?.uid == n?.uid)`, a second subscription with the same body and type change. Every input it needs is an owner transition, which `userChanges()` carries. It never needed refreshes, because `distinct` by uid already dropped them |
| 141 `idTokenChanges()`, Siri only | Replay on listen and every refresh, to keep the native Keychain mirror current. Its `user` carries the token through `getIdTokenResult()` | Split by event kind. (a) Owner transitions, including the cold-restore replay, already reach Siri through the 116 listener: `SiriIntegration.accountChanged(user)` (`siri_integration.dart:527`, now taking `AuthUserSnapshot?`) ends in `refreshSession(user, AuthService.instance.currentAccessToken)` at 575 for a non-null owner, so the mirror is published without waiting for a refresh. (b) Refreshes: the 141 subscription becomes `tokenChanges().listen(...)`, which calls `refreshSession(AuthService.instance.currentUser!, event.token)` only when `event.ownerUid == AuthService.instance.currentUser?.uid`, and otherwise drops the event |

- `AuthTokenRefreshed` is a new value type beside `RefreshedAuthToken` in
  `services/auth/auth_token_result.dart`:
  `AuthTokenRefreshed({required String ownerUid, required RefreshedAuthToken token})`.
  `RefreshedAuthToken` and the `forceRefresh()` return type stay unchanged,
  so `AuthTokenGateway` and its 25 fakes do not change.
- `AuthService.currentAccessToken` returns the cached `RefreshedAuthToken` of
  the current session, or `null`. It reads the gateway's in-memory session and
  does no network call.
- Late refreshes are fenced twice. Inside `CognitoAuthSessionGateway`, each
  grant captures the session epoch it started from, and a response that
  arrives after `signOut()` or a new `signIn()` bumped the epoch is discarded
  before it is stored or emitted. In Dart, the Siri listener compares
  `ownerUid`, and `SiriIntegration.refreshSession` keeps its own
  `_accountGeneration` and `_uid` checks (`siri_integration.dart:707-740`).
- A terminal refresh failure clears the session in the gateway and emits
  `null` on `userChanges()`, as Firebase emits `null` on revocation today.
  `AuthService.sessionExpiredEvents` still fires, so the 168-180 listener
  keeps its reauthentication prompt.
- `AuthService.handleAuthUserChanged(uid)` (384) is still called from both
  owner listeners, so `_invalidateRefreshes` bumps `_sessionGeneration` on
  every owner change, as it does today.

**Tests (`test/unit/auth_listener_migration_test.dart`).** A fake
`AuthSessionGateway` with a broadcast controller per stream:

1. **Cold restore before listener attachment.** `restore()` completes with
   owner `A` before `AuthProvider` subscribes. On subscription, both owner
   listeners run once with `A`, the token check calls `getIdToken()` when the
   cached token is empty, and Siri's `refreshSession` receives `A` and the
   restored token, with no refresh event emitted.
2. **Sign-out without refresh.** With owner `A` signed in, `signOut()` emits
   `null` on `userChanges()` and nothing on `tokenChanges()`. The test asserts
   that `SharedPreferencesUtil().authToken` is empty,
   `tokenExpirationTime == 0`, `authToken` is `null`, and
   `SiriIntegration.accountChanged(null)` ran.
3. **A previous owner's late refresh.** A refresh starts for `A`, then the
   user switches to `B`. The fake completes `A`'s grant after the switch. The
   gateway test asserts nothing is stored or emitted. The provider test
   injects a stale `AuthTokenRefreshed(ownerUid: 'A', ...)` directly and
   asserts that Siri's mirror still holds `B`'s token.
4. **Refresh for the current owner.** A refresh for `B` calls
   `refreshSession` once with the new token. Neither owner listener runs and
   `handleAuthUserChanged` is not called.
5. `onboarding_sync_runtime` and `on_device_background` receive the replayed
   owner on subscription, and `on_device_background` cancels its runner when
   `userChanges()` emits a different uid.

### Client: Firebase retirement

Removing the Firebase packages breaks compilation unless every owner that
imports them or calls them at startup moves in the same change. The census
below is every `package:firebase_*`, `google_sign_in` and `sign_in_with_apple`
import in `app/lib/`, every Firebase symbol use outside those imports, and every
native and build file that links a Firebase SDK. The `firebase_auth` consumers
are in the table above. Paths are relative to `app/`.

**Startup (`lib/main.dart`).**

| Lines | Today | Port |
| --- | --- | --- |
| 24-27, 53-54, 58 | imports `firebase_auth`, `firebase_core`, `firebase_crashlytics`, `firebase_messaging`, `firebase_options_local.dart`, `firebase_options_prod.dart`, `startup_firebase.dart` | removed |
| 120 | imports `utils/debugging/crashlytics_manager.dart` | imports `utils/debugging/crash_reporter.dart` |
| 131-147 | `_firebaseOptionsForFlavor`, `_ensureFirebaseApp` | deleted |
| 150-154 | `onDeviceOverflowBackground` initializes Firebase, then Opus | keeps the `vm:entry-point`. Its `initialize` calls `AuthService.instance.restoreSession()`, which delegates to the held gateway's `restore()` (reads `flutter_secure_storage`, no network) instead of `_ensureFirebaseApp`, because `on_device_background.dart:35,49` now asks the gateway for the owner |
| 157-196 | `_firebaseMessagingBackgroundHandler(RemoteMessage)` | deleted. With no FCM there is no background data message. The handlers it called (`ActionItemNotificationHandler`, `MergeNotificationHandler`, `ImportantConversationNotificationHandler`, `ChatAnswerNotificationHandler`) stay, because the foreground paths and tests use them |
| 214-245 `_reportBootCrash` | journals breadcrumbs, then returns if `Firebase.apps.isEmpty`, else writes to `FirebaseCrashlytics` | journals breadcrumbs, then calls `CrashReporter.instance.recordBoot(kind, error, stack, details: details, fatal: fatal, origin: errorOrigin)`. The `Firebase.apps` guard goes |
| 270 | `startupStage('firebase_init', _ensureFirebaseApp)` | deleted. The `auth_restore` stage that `CognitoAuthSessionGateway` adds runs in the same position, before `BootCrashHandlers.install` |
| 272-300 | `BootCrashHandlers.install(initialize: CrashlyticsManager.init, ...)` | `initialize: CrashReporter.init`. Handler order is unchanged, so `test/unit/boot_crash_handlers_test.dart` keeps passing |
| 303-308 | Firebase Auth emulator stage | deleted |
| 312 | `NotificationService.instance.initialize` | unchanged call. The instance is the local service below |
| 314-317 | `FirebaseMessaging.onBackgroundMessage(...)` behind `isFCMSupported` | deleted. `PlatformManager.isFCMSupported` (`utils/platform/platform_manager.dart:99`) is deleted with it; it has no other reader |
| 408 | `CrashlyticsManager.init` under physical qualification | `CrashReporter.init` |
| 590-591 | AwesomeNotifications action listener and isolate port | unchanged. Local notification taps keep routing through `NotificationUtil.onActionReceivedMethod` (`services/notifications.dart:42`) to `handleFcmDataTap` |

**Notifications.** `services/notifications/notification_service.dart` imports
`notification_service_fcm.dart` unconditionally (line 5), and that file imports
`firebase_messaging` (line 10). Both move:

- New `services/notifications/notification_service_local.dart` with
  `_LocalNotificationService implements NotificationInterface` and
  `createNotificationService()`. `notification_service.dart` imports it in
  place of the FCM file, and its header comment says local-only.
- These members move from `_FCMNotificationService` unchanged: `initialize`
  (locale load, channel, `_initializeAwesomeNotifications`, badge reset),
  `showNotification`, `createNotification`, `clearNotification`,
  `hasNotificationPermissions` and `getTimeZone`. Every local producer
  (`sync_health_watchdog.dart:157`, `device_provider.dart:649,731`,
  `user_provider.dart:484-565`) keeps its behaviour.
- `requestNotificationPermissions` keeps the AwesomeNotifications permission
  request and drops the `register()` side effect.
- `register`, `saveFcmToken`, `saveNotificationToken` and `listenForMessages`
  return immediately. Their callers (`app_shell.dart:453`, `home/page.dart:160-161`,
  `auth_provider.dart:345`, `onboarding_provider.dart:233`) are unchanged.
- `listenForServerMessages` returns a broadcast stream that never emits, so
  Home's subscription (`home/page.dart:818`) stays valid.
- `notification_service_fcm.dart` is deleted. `fcm_token_registration.dart`
  has no Firebase import and stays, because `pages/settings/sign_out.dart:9`
  and `backend/http/api/notifications.dart:6` import its order key.
  `test/services/fcm_registration_admission_test.dart` is deleted with the
  FCM service.

This is what "NotificationService becomes local-only" means in the cut list:
no push token is read or sent, and local notifications still show.

**Crash and log sink.** `utils/debugging/crashlytics_manager.dart` and
`utils/logger.dart:4` import `firebase_crashlytics`. The port renames the
manager to `CrashReporter` in `utils/debugging/crash_reporter.dart` and keeps
its public members (`init`, `applyBuildProvenanceKeys`, `identifyUser`,
`logInfo`/`logError`/`logWarn`/`logDebug`/`logVerbose`, `setUserAttribute`,
`setEnabled`, `reportCrash`, `reportNonFatal`, `getNavigatorObserver`,
`isSupported`), so its callers change only the type name:
`platform_manager.dart:14,34`, `analytics_manager.dart:39,119-234`, `main.dart`,
`test/unit/physical_qualification_test.dart`,
`test/unit/build_provenance_registration_test.dart` and
`integration_test/perf/production_shell.dart`.

- The MVP sink is local. `reportCrash`, `reportNonFatal` and `recordBoot`
  append the error, stack, attributes and build provenance as one JSON line to
  a new bounded file, `crash_reports.jsonl` in the app documents directory
  (newest 200 entries; the writer drops the oldest line past that). They also
  call `DebugLogManager.logError` (`utils/debug_log_manager.dart:166`), which
  writes only when the user has turned on developer logs (`_append`, 99).
  `DebugLogManager.listLogFiles` (115), which today matches only
  `omi_debug_*.log`, also returns `crash_reports.jsonl`, so the
  developer settings "share logs" action (`pages/settings/developer.dart:86`)
  attaches it. `log*` calls append
  to `BootJournal`'s breadcrumbs. `setUserAttribute` and `identifyUser` keep a
  bounded in-memory map that the next report includes. `isSupported` is
  `false`.
- `reportNonFatal` keeps `recordNonFatalFailure`
  (`utils/debugging/non_fatal_reporter.dart`, no Firebase import), so a sink
  failure still cannot escape into the global error handler.
- `logger.dart`: `CrashlyticsTalkerObserver` becomes `CrashReporterTalkerObserver`,
  which calls `CrashReporter.instance.reportNonFatal`. The Crashlytics import goes.
- `test/support/crashlytics_recorder.dart` (a Crashlytics method-channel fake
  used by `capture_provider_test.dart` and `force_processing_placeholder_test.dart`)
  becomes `crash_reporter_recorder.dart`, a fake `CrashReporter` sink with the
  same `recordErrors` assertions.
- Rejected: keeping Crashlytics. It needs `firebase_core` and a Firebase
  project, which this port removes. A remote crash sink on AWS is follow-up
  work and plugs in behind `CrashReporter` without touching its callers.

**Environment and qualification.**

- `env/env.dart:10-15,139` (`firebaseAuthEmulatorHost`/`Port`) and `166-172`
  (`validateFirebaseProject`) are deleted. `env/environment_profile.dart`
  drops `firebaseProjectId` and `usesFirebaseAuthEmulator`, and
  `test/unit/env_test.dart:56,63,77` drops those assertions.
- Physical qualification, the hermetic journeys, the simulator lane and the
  session harness sign in through the local issuer instead of the Auth
  emulator. "Client: non-production auth paths" below owns every one of those
  entrypoints, including `env/physical_qualification.dart` and
  `ios/Runner/AppDelegate.swift:302-304`.

**iOS Siri (native).** `ios/Runner/SiriIntegration/SiriSession.swift`,
`SiriBridge.swift` and `SiriDebugProbe.swift` import `FirebaseAuth` and
`FirebaseCore`. They authorize each Siri request by reading
`Auth.auth().currentUser` and fetch a fresh token with `getIDToken`
(`SiriSession.swift:119-190`), and `SiriNativeAuthFence` listens to Firebase
auth state (`SiriBridge.swift:262-300`). The census keeps Siri App Intents in
the MVP, so the native owner check moves onto the session mirror that Dart
already publishes (`siri_integration.dart:707-742`, `publishSessionConfig`):

- `refreshSession` takes an `AuthUserSnapshot` and a `RefreshedAuthToken` in
  place of the Firebase `User` and its `getIdTokenResult()`. It runs on every
  owner transition and on every current-owner refresh, so the Keychain mirror
  always holds the newest access token and its `exp`. Its callers move with
  it: `auth_provider.dart:141-143`, which becomes the `userChanges()` plus
  owner-checked `tokenChanges()` pair in "Auth listener migration", and
  `siri_integration.dart:575`, which passes `AuthService.instance.currentUser`
  and `AuthService.instance.currentAccessToken`.
- `requireFirebaseOwner` becomes `requireMirroredOwner(uid)`: the current
  config's uid must equal `uid` and the mirrored token must exist. It keeps the
  protected-data branch (`setAuthResolutionPending`).
- `token(for:)` returns the mirrored token while it has more than 60 s left,
  the same expiry check it already applies to the fallback path (176-178).
  Otherwise it throws `Failure.auth`, and the intent asks the user to open Omi.
  The refresh token is not mirrored to native code.
- `SiriNativeAuthFence` is deleted. Dart already calls `accountChanged` and
  `prepareForSignOut` (`siri_integration.dart:492-527`) on sign-out and
  account switch, and it now drives them from `userChanges()`.
- `SiriDebugProbe.swift`'s Firebase Auth-emulator probes (the
  `FirebaseApp.configure` blocks at 301, 966 and 1022) are deleted. They
  compile only under `OMI_SIRI_PROBE`.
- Accepted gap: a Siri request more than 60 minutes after the app last
  refreshed its token fails until the app is opened. Rejected: mirroring the
  refresh token into the Keychain for native refresh. It widens where a
  30-day credential lives for a convenience feature.

**Build files and config.**

- `pubspec.yaml`: remove `firebase_core`, `firebase_auth`, `firebase_messaging`,
  `firebase_crashlytics`, `google_sign_in` and `sign_in_with_apple` (40-47),
  `google_sign_in_ios` (161), and the dev dependencies
  `firebase_core_platform_interface` (178) and
  `firebase_messaging_platform_interface` (197). The
  `enable-swift-package-manager: false` setting (209-215) stays; its Firebase
  reason goes, and re-enabling SPM is a separate change.
- Dart files deleted: `startup_firebase.dart`, `firebase_options_local.dart`,
  the generated `firebase_options_prod.dart` (gitignored),
  `utils/debugging/crashlytics_manager.dart`,
  `services/notifications/notification_service_fcm.dart`,
  `test/unit/startup_firebase_duplicate_app_test.dart` and
  `test/unit/firebase_local_options_test.dart`.
- Android: the Google services and Crashlytics plugins
  (`android/settings.gradle:23-24`, `android/app/build.gradle:4-5`), the
  `com.google.firebase` keep and dontwarn rules (`android/app/proguard-rules.pro:50-55`),
  the `FirebaseInitProvider` removal in
  `android/app/src/heavyPerf/AndroidManifest.xml:5-6` (nothing to remove once
  the SDK is gone), and the copied `google-services.json` files.
- iOS: both "Copy Google Plist file" build phases and every
  `GoogleService-Info.plist` reference in `ios/Runner.xcodeproj/project.pbxproj`
  (26, 76, 434, 656, 997, 1031, 1055, 1106), and the plist files under
  `ios/Config/{Dev,Prod}/`. `ios/ImageNotification/NotificationService.swift`
  imports `FirebaseMessaging` but no Xcode target builds it; delete the
  directory.
- Setup scripts: the Firebase copy steps in `setup.sh:111-146`, `test.sh:42-52`,
  `setup/scripts/setup.ps1:50-58` and `setup-personal.sh:19-53`, and the
  `setup/prebuilt/` and `setup/release-firebase/` Firebase files.
- Tests that import Firebase test interfaces move to an `AuthSessionGateway`
  fake: `test/services/siri_integration_test.dart`,
  `test/utils/auth/sign_in_failure_test.dart`,
  `test/unit/authenticated_request_401_test.dart`,
  `test/unit/auth_service_token_result_test.dart`,
  `test/widgets/home_calendar_filter_test.dart`,
  `test/widgets/home_announcement_timer_test.dart`,
  `test/widgets/home_product_mounted_guard_test.dart`,
  `test/providers/device_provider_fully_charged_alert_test.dart` and
  `test/providers/device_provider_disconnect_telemetry_test.dart`.
- Tests and harnesses that construct `FirebaseAuthException` to script a
  refresh outcome throw `AuthRefreshException` instead ("Refresh failures"
  below): `integration_test/journeys/support/hermetic_boot.dart`,
  `test/unit/auth_service_token_result_test.dart`,
  `test/unit/authenticated_request_401_test.dart`,
  `test/unit/token_refresh_loop_test.dart` and
  `test/utils/auth/sign_in_failure_test.dart`.
- `test/unit/auth_provider_link_session_test.dart` is deleted. Its five cases
  cover Google and Apple provider linking and credential collisions, which go
  with the link paths, and it is the only caller of the `firebaseAuth` and
  `authorizeProviderLink` parameters of `AuthService.forTesting`.
- `integration_test/perf/production_shell.dart` imports `firebase_auth`,
  `firebase_core`, `firebase_crashlytics`, `firebase_messaging` and
  `firebase_options_local.dart` (18-21, 46). It is generated from `main.dart`
  by `app/tool/perf_shell.py`. The auth card regenerates it after the
  `main.dart` changes, and `python3 app/tool/perf_shell.py --check` (run by
  `app/test.sh:15` and the `mobile-perf-shell-fidelity` entry of
  `.github/checks-manifest.yaml`) passes on the result.

**Compile and startup checks (auth card).**

1. No Firebase reference is left. `grep -rE "package:(firebase_|google_sign_in|sign_in_with_apple)" app/lib app/test app/integration_test`,
   `grep -rE "import Firebase|FirebaseApp|FIRMessaging" app/ios app/macos` and
   `grep -rE "google-services|firebase" app/android --include=*.gradle --include=*.pro`
   each return nothing, and `flutter pub deps --style=list` lists no
   `firebase_` package.
2. `flutter analyze` is clean, and `flutter build apk --flavor dev` and
   `flutter build ios --flavor dev --no-codesign` both succeed with the fork's
   dart-defines.
3. A host test runs `_init()` with a fake `AuthSessionGateway`, the
   AwesomeNotifications platform fake, and no Firebase method-channel mocks.
   It asserts that `BootJournal` records every stage through
   `service_manager_init` and records no `firebase_init` stage.
4. A host test of `_LocalNotificationService`: `initialize` creates the
   `channel` channel. `createNotification` presents when permission is granted
   and returns false when it is not. `register` and `saveFcmToken` make no HTTP
   call (no `POST /v1/users/fcm-token`). `listenForServerMessages` emits
   nothing.
5. A host test throws from an `_init` stage and asserts that
   `CrashReporter`'s recording fake receives one fatal boot report, and that
   `crash_reports.jsonl` holds it with developer logs off.
6. On an Android device and an iOS device: cold start reaches Home signed in.
   A sync-health or fully-charged local notification shows, and tapping it
   opens its route. A Siri "add a memory" intent succeeds within an hour of
   opening the app, and asks to open Omi after the mirrored token expires.

**Acceptance check for the auth card:** checks 1-6 pass, and sign-in, cold
restore, sign-out and account switch work against the pool.

### Client: non-production auth paths

Five entrypoints sign in without a person at the hosted UI, and each one
depends on the Firebase Auth emulator or on Firebase exception types today:

- **Physical qualification.** `main.dart:350-359` calls
  `signInWithLocalDevToken(uid: fixtureUid)` under
  `OMI_PHYSICAL_QUALIFICATION`. `PhysicalQualification.install`
  (`env/physical_qualification.dart:126-158`) requires a private emulator host
  and the three native Firebase collection flags. The fixture backend
  (`scripts/dev-harness/physical_capture_fixtures/server.py`) mints an
  unsigned custom token at `/v1/auth/local-dev/custom-token` (218) and checks
  each bearer token through the emulator's `accounts:lookup` (179-191).
- **Hermetic journeys.** `integration_test/journeys/support/hermetic_boot.dart`
  installs token-only fakes through `installLocalHarnessTokenGateway`, and
  `ScriptableGateway.forceRefresh` throws `FirebaseAuthException` to script
  terminal and transient outcomes (188-200). j4 asserts the local-dev re-mint
  reaches `POST /v1/auth/local-dev/custom-token`
  (`j4_expired_session_recovery_test.dart:52-63`).
- **Simulator lane.** `run_journeys.sh --lane simulator` boots the real app
  against the session harness. The journeys README (60, 65) says it uses real
  FirebaseAuth against the Auth emulator.
- **Session and memory harness.** `dev_harness/config.py:368-372` starts the
  backend with `FIREBASE_AUTH_EMULATOR_HOST`, `cli.py:664-684` starts the Auth
  emulator, `mobile_fixtures.seed_synthetic_user` (113) posts to the custom-token
  route, and `memory_scenarios.py:995-1012` and
  `chat-first-e2e-fixture.sh` sign in with `accounts:signInWithPassword`.
- **Developer sign-in.** `pages/onboarding/auth.dart:122-130` shows a
  local-development button that calls `AuthProvider.onLocalDevSignIn`
  (`providers/auth_provider.dart:210`).

**Decision.** Replace the Auth emulator with a local issuer inside the
backend, and give the app a second `AuthSessionGateway` implementation that
talks to it. Every non-production entrypoint then signs in through the same
gateway contract as production, the backend verifies through the same
`verify_token` and the same failure mapping, and no Firebase package or
emulator is left in any lane.

**Rejected.**

- Keeping the Auth emulator for test lanes. It keeps `firebase_auth` in the
  app, which the retirement removes, and its tokens are Firebase-shaped, so
  `verify_token` would need a Firebase branch kept alive only for tests.
- A Cognito emulator (`cognito-local`, moto). Neither is in either requirements
  file, and their hosted-UI and token-endpoint behaviour is not the pool's.
- A real test user in the pool. It needs network, which breaks
  qualification's private-literal isolation, puts a password in every harness,
  and adds a user to a pool shared with other `sauhsoj.wtf` apps.
- `ADMIN_KEY` from the app. It is an impersonation credential, and the fork
  runs with `ADMIN_KEY_AUTH_ENABLED=false`.

**Local issuer contract (backend).** New `utils/other/local_auth_issuer.py`
and `routers/local_auth.py`, mounted at `/_local/auth` beside the
`/_local/storage` mount (`main.py:110-113`). `cryptography` and `PyJWT` are
already in `backend/requirements.txt` and `backend/pusher/requirements.txt`.

- **Gate.** The router mounts only when `OMI_LOCAL_AUTH_SIGNING_KEY_PATH` is
  set and `utils/env_loader.py::stage_from_env` (81) returns `local` or
  `offline`. The harness sets those stages (`dev_harness/config.py:402,426`).
  The fork deploy runs `OMI_ENV_STAGE=dev` ("Runtime settings"). If the path
  is set under any other stage, or the key file is readable by group or
  other, `main.py` raises at import, so the task fails its health check
  instead of serving a token mint. Without the path the route does not exist
  and answers 404.
- **`POST /_local/auth/token`**, form-encoded, in the Cognito
  `/oauth2/token` response shape so the client parses one shape:
  - `grant_type=urn:omi:local-dev`, `uid` matching `^[A-Za-z0-9._-]{1,128}$`,
    optional `email`. Returns `access_token`, `refresh_token`,
    `expires_in: 3600`, `token_type: Bearer`, plus `uid` and `email`.
  - `grant_type=refresh_token`, `refresh_token`. Returns a new access token.
  - Any other grant, a malformed uid, or a refresh token that fails
    verification answers 400 `{"error": "invalid_grant"}`, the code Cognito
    sends for a revoked or expired refresh token.
- **Tokens.** RS256, signed with the key at the path. The access token has
  `iss: urn:omi:local-dev-issuer`, `token_use: access`,
  `client_id: omi-local-dev`, `sub: <uid>` and a one-hour `exp`. The refresh
  token is a JWT with `token_use: refresh` and a 30-day `exp`, so the issuer
  holds no state and works across processes. `verify_token` refuses it as a
  bearer token because `token_use` is not `access`.
- **Verifier.** `verify_token` reads the unverified `iss`. The pool issuer
  takes the Cognito path. `urn:omi:local-dev-issuer` is accepted only while
  the same gate holds, against the public half of the configured key, so it
  never fetches a key set and never raises `AuthProviderUnavailable`. Under
  any other stage a local-issuer token is `AuthCredentialInvalid`. Every
  failure goes through the "Verification failures" mapping unchanged.
- **Route policy.** The router is not mounted under the hermetic environment
  that `scripts/route_policy_inventory.py` builds (it never sets the key path),
  so it needs no manifest entry. The `/v1/auth/local-dev/custom-token` entry
  (`route_policy_manifest.yaml:3560`) is deleted with that route.

**Client gateway.** New `services/auth/local_issuer_gateway.dart`,
`LocalIssuerAuthSessionGateway implements AuthSessionGateway`:

- `signInLocalDev({required String uid, String? email})` runs the
  `urn:omi:local-dev` grant against `${Env.apiBaseUrl}_local/auth/token` and
  stores the session like the Cognito gateway does. `signIn()` calls it with
  the default uid `local-dev-user`. The uid and email come from the response
  body. The gateway never parses the access token, so a fixture server may
  issue opaque tokens.
- `forceRefresh()` runs the refresh grant. Its failures follow "Refresh
  failures" below, so `invalid_grant` is terminal.
- `signOut()` wipes local state. There is nothing to revoke.
- **Selection.** A new define, `OMI_AUTH_GATEWAY`, takes `cognito` (the
  default) or `local_issuer`. The profile cannot select it: the fork's own dev
  build is a `dev` flavor, `AppEnvironmentProfile.forFlavor` gives that the
  `local_dev` profile when `OMI_APP_PROFILE` is unset (`environment_profile.dart:56-59`),
  and `Env.validateProfilePairing` (`env.dart:156`) requires `local_dev` for
  every non-prod flavor. `validateProfilePairing` also refuses `local_issuer`
  unless the flavor is `dev`, the profile is `local_dev` and the API host is a
  loopback or private IPv4 literal (`PhysicalQualification.isPrivateLiteral`,
  113, plus loopback).
- `AuthProvider.isLocalDevProfile` (`auth_provider.dart:208`) becomes
  `usesLocalIssuer` (`Env.authGateway == local_issuer`). Keyed on the profile,
  it would hide the hosted-UI button in the fork's own dev build.
  `pages/onboarding/auth.dart:54,89` show the hosted-UI button when it is
  false, and 122-130 show the local-development button when it is true.
  `onLocalDevSignIn` calls `signInLocalDev()`.
- `installLocalHarnessTokenGateway(AuthTokenGateway)` (`auth_service.dart:136`)
  keeps its signature and both assertions, and wraps the fake with
  `asAuthSessionGateway`, the same adapter `AuthService.forTesting` uses
  ("Token-only injection seams").

**Refresh failures.** New `services/auth/auth_refresh_exception.dart`:
`AuthRefreshException(String code, {required bool terminal})`. Both gateways
throw it from `forceRefresh()`:

| Refresh response | `code` | `terminal` |
| --- | --- | --- |
| 400 or 401 with an RFC 6749 §5.2 `error` body (`invalid_grant`, `invalid_client`, `unauthorized_client`, `invalid_request`, `unsupported_grant_type`) | that `error` value | true |
| 429 or any 5xx | `http_<status>` | false |
| socket or TLS error | `network_error` | false |
| a body that is not the expected JSON | `malformed_response` | false |

A timeout keeps today's `TimeoutException` path (`refresh_timeout`).
`AuthService.refreshIdToken` replaces `on FirebaseAuthException` (578-586)
with `on AuthRefreshException`: terminal clears cached auth and returns
`AuthTokenTerminalFailure(code)`, transient returns
`AuthTokenTransientFailure(failureClass: 'refresh_transient', code)`.
`_terminalTokenErrorCodes` (156) is deleted. The telemetry value
`firebase_transient` becomes `refresh_transient`; whether any dashboard keys
on the old value is UNVERIFIED.

`_scheduleLocalDevRecovery` (453-480) and its call in `getIdToken` (423) are
deleted. It worked around `forceRefresh` hanging against the Auth emulator by
minting a new session instead. Against the local issuer a refresh is a grant
on the same backend that would mint, so the re-mint adds no path the refresh
lacks. A transient failure now behaves as in production: the caller gets
`null` and the next request refreshes again.

**Physical qualification.** The fixture server is the whole backend in this
lane and stays stdlib-only:

- `server.py` implements `POST /_local/auth/token` for the fixture uid only
  (any other uid answers 400 `invalid_grant`). It issues opaque access and
  refresh tokens from `secrets.token_urlsafe(32)` and keeps them in memory.
  `authenticated()` compares the bearer value with `hmac.compare_digest`
  against the issued access tokens. `custom_token`, `emulator_opener`, the
  `/v1/auth/local-dev/custom-token` branch, `--auth-port` (435), `auth_url`
  (441) and the manifest's `auth_emulator_url` (451, 462) are deleted.
- `main.dart:350-359` reads the restored principal from the gateway's
  `currentUser` instead of `FirebaseAuth.instance.currentUser`, keeps the
  different-principal refusal, and calls
  `signInLocalDev(uid: PhysicalQualification.fixtureUid)` inside the same
  `fixture_signin` stage.
- `PhysicalQualification.install` (126-158) drops the emulator host from the
  private-literal check (137) and the HTTP allowlist (146), so the allowlist
  holds only the API host. It drops the three native Firebase flags from the
  isolation check (152-154) and keeps the bundle-id check. It also requires
  `Env.authGateway == local_issuer`. `ios/Runner/AppDelegate.swift:302-304`
  stops reporting the flags, and `ios/test/check_userdefaults_plist_values.py:17-19`
  follows.
- `physical_capture_build.py` sets `OMI_AUTH_GATEWAY=local_issuer` and drops
  `--auth-host`/`--auth-port` (173-174), `private_host(args.auth_host)` (192),
  the Firebase and `OmiPhysicalAuthEmulator*` plist keys (257-263, which no
  native code reads), the emulator defines (283-284) and the two `firebase_*`
  entries of `EXCLUDED_PLUGINS` (23-24), whose packages no longer exist.
- `physical_capture_collect.py` (185, 458) and `physical_capture_verify.py`
  (107) drop `auth_emulator_url` from the run manifest. The fixtures in
  `physical_capture_test.py:31` and `physical_capture_collect_test.py:32,63`
  follow, as do `physical_capture.md`, `PHYSICAL_CAPTURE_BUILD.md` and
  `physical_capture_fixtures/README.md`.

**Hermetic journeys.**

- `hermetic_boot.dart` drops its `firebase_auth` import (4). `_SyntheticGateway`
  and `ScriptableGateway` still implement `AuthTokenGateway` and install
  through the unchanged seam. `ScriptableGateway` throws
  `AuthRefreshException(code, terminal: true)` for a terminal outcome and
  `AuthRefreshException('network_error', terminal: false)` for a transient
  one. Its doc comment (166-169) names the gateway contract, not Firebase.
- j4's terminal leg scripts `invalid_grant` instead of `user-token-expired`
  (71) and keeps its assertions: the session expires and the next request is
  blocked. The re-mint leg (34-63) becomes "a transient refresh failure
  returns `null` without expiring the session, and the next request refreshes
  and succeeds". The negative test (99-116) becomes "a transient failure
  never expires the session on any profile", because no profile re-mints.
- `support/fixture_backend.dart:187-192` drops the custom-token case.
- `journeys/README.md` j4 row (35) and boundary row (60) describe the
  gateway contract, and its lane note (65) says the simulator lane uses the
  local issuer. `run_journeys.sh:14` says the same.
- `test/unit/local_dev_session_recovery_test.dart` and
  `test/unit/auth_service_local_dev_test.dart` are deleted with the recovery
  and `signInWithLocalDevToken`.

**Simulator lane and session harness.** The simulator lane stays, with the
local issuer in place of the Auth emulator. The Firestore emulator in this
lane is a data-store question, owned by the document-store adapter card, and
is out of scope here.

- `dev_harness/config.py:368-372` drops `FIREBASE_AUTH_EMULATOR_HOST`,
  `FIREBASE_AUTH_PROJECT_ID` and `FIREBASE_API_KEY` and sets
  `OMI_LOCAL_AUTH_SIGNING_KEY_PATH`. `dev-up` writes a fresh RSA-2048 key at
  `<state_root>/services/auth/local-issuer.pem` with mode 0600. `AUTH_PORT`
  (15) and `LOCAL_FIREBASE_API_KEY` (26) go.
- `cli.py`: `_firebase_command` (664-684) configures only the Firestore
  emulator, the doctor's Java and Node checks (362-383) name only Firestore,
  and the status line (433) prints the issuer URL.
- `mobile_fixtures.py`: `LOCAL_DEV_CUSTOM_TOKEN_PATH` (28) becomes
  `LOCAL_AUTH_TOKEN_PATH = "/_local/auth/token"`, and `seed_synthetic_user`
  (113) runs the `urn:omi:local-dev` grant. Its receipt keeps the same fields
  and still never records a token. `tests/test_mobile_fixtures.py` follows.
- `mobile_session.py:825` and `session_evidence.py:313` replace
  `auth_emulator_host` with `auth_issuer_url`, which must be loopback. That
  changes a required field of `contracts/session/session-evidence-v1.schema.json`
  (164-167), so the change publishes `session-evidence-v2.schema.json` and
  moves every file that names v1 to it (`git grep -l session-evidence-v1`).
  Which of those read the endpoints block is UNVERIFIED.
  `tests/test_session_evidence.py`, `tests/test_mobile_session.py`,
  `tests/test_cli.py`, `tests/test_env_stage.py` and
  `tests/spine/test_live_session.py:80` follow.
- `memory_scenarios.py`: the `auth` seed operations (`_auth_seed`, 190) go.
  The issuer accepts the scenario's own uids, so `_resolve_auth_uid_map`
  (1011) becomes the identity map and `_lookup_auth_uid` (995) is deleted.
  `canonical-auth-uids.json` is still written, so
  `backend/config/chat_first_e2e_fixture.py` and `chat-first-e2e-fixture.sh`
  keep reading it. The script gets its token from the `urn:omi:local-dev`
  grant instead of `accounts:signInWithPassword`.
- `backend/testing/referral_campaign` verifies the app's real token (its
  `server.py:14` refuses to start without both emulators). It requires
  `OMI_LOCAL_AUTH_SIGNING_KEY_PATH` in place of the Auth emulator.
  `replay_harness_phase0a/runner.py:143` and `sync_cloud_tasks_stack/run.py:272`
  authenticate with `ADMIN_KEY` and set the emulator variable only to start
  `firebase_admin` offline, so they drop it.

**Every Auth-emulator owner.** `git grep -il "auth emulator\|FIREBASE_AUTH_EMULATOR\|firebaseAuthEmulator\|local-dev/custom-token\|signInWithLocalDevToken\|auth_emulator"`
outside `desktop/` and these ADR files lists 64 files on `a04153d180`. Each
one has a port above or one of these dispositions:

| Owner | Disposition |
| --- | --- |
| `backend/main.py:87-99`, `backend/desktop_backend.py:59-64` | The emulator branch goes with `firebase_admin.initialize_app` ("Auth: Cognito access token"). `tests/unit/test_agent_vm_firebase_project_split.py:45` loses its emulator case |
| `backend/utils/env_loader.py:121-140,188` | `_skip_env_key_for_auth_emulator` and the credential pop are deleted. `tests/unit/test_env_loader.py` drops those cases |
| `backend/routers/auth.py:1271-1340` (`local_dev_custom_token`) | Deleted, with `tests/unit/test_local_dev_auth.py`. The rest of the router is cut by `OMI_DISABLED_ROUTERS=auth` |
| `app/setup.sh:335,578`, `app/setup/scripts/setup.ps1:133`, `app/scripts/profile_flutter_android.sh:141` | Pass `--dart-define=OMI_AUTH_GATEWAY=local_issuer` instead of the emulator host |
| `app/README.md:30,147`, `docs/doc/developer/AppSetup.mdx`, `docs/doc/developer/backend/Backend_Setup.mdx`, `backend/docs/runbooks/local-emulator-manual-qa.md`, `scripts/dev-harness/MEMORY_SCENARIOS.md`, `MOBILE_SESSIONS.md` | Describe the local issuer. The upstream-facing developer docs change only on the fork branch |
| `app/lib/startup_auth.dart:10`, `test/unit/auth_refresh_timeout_test.dart:15`, `test/unit/startup_auth_timeout_test.dart:10` | Comments that cite the emulator measurement behind the 8 s refresh ceiling. The ceiling stays, and the comments say "token refresh" |
| `app/integration_test/ios_auth_refresh_canary_test.dart:13,38,42,46` | Reason strings say "persisted session" and "forced refresh" instead of Firebase |
| `app/ios/Runner/SiriIntegration/SiriDebugProbe.swift` | Covered in "iOS Siri (native)" |
| `scripts/dev-harness/dev_harness/desktop_profile.py`, `desktop-run-local.sh`, `verify-desktop-local-launch.sh`, `tests/test_desktop_profile.py`, and the `dev-desktop`, `dev-verify` and `desktop-run-local` Makefile targets | Deleted. The desktop apps are outside the fork's MVP, and `desktop_profile.py` reads the emulator host that `config.py` no longer has. `desktop/` itself is not built by the fork and is untouched |
| `backend/scripts/firebase_release_probe_token.py`, `smoke_mcp_chatgpt_review.py`, `sync_firebase_google_provider_secret.py` and their tests | Upstream release and ops tooling against Firebase projects. Not run by the fork and not imported by the serving path. Left unchanged; `firebase-admin` stays installed |

**Tests and acceptance (auth card).**

1. Backend unit tests for the issuer: both grants return the Cognito response
   shape. A bad grant, a malformed uid, a refresh token with a bad signature
   and an access token presented as a refresh token each answer 400
   `invalid_grant`. The router is absent without the key path. Import raises
   with the key path under `OMI_ENV_STAGE=dev` and with a mode-0644 key.
2. `test_auth_tokens.py` gains: a local-issuer token verifies under
   `local`/`offline` with the key path; the same token is
   `AuthCredentialInvalid` under `dev` and without the key path; a local
   refresh token is `AuthCredentialInvalid` as a bearer token.
3. App unit tests: `local_issuer_gateway_test.dart` covers both grants and
   every row of the refresh-failure table. `env_test.dart` covers
   `validateProfilePairing` refusing `local_issuer` with the prod flavor and
   with a public API host. `auth_service_token_result_test.dart` asserts
   terminal and transient `AuthRefreshException` map to
   `AuthTokenTerminalFailure` and `AuthTokenTransientFailure`.
4. `bash integration_test/journeys/run_journeys.sh` passes all five journeys
   and their negatives on the hermetic lane, and the simulator lane passes
   against `make dev-up`.
5. `python3 app/tool/perf_shell.py --check` passes.
6. `python3 -m pytest scripts/dev-harness/physical_capture_test.py
   scripts/dev-harness/physical_capture_collect_test.py
   scripts/dev-harness/tests` passes.
7. Physical qualification compiles: `physical_capture_build.py --prepare-only`
   succeeds, and `flutter build ios --flavor dev --profile` with
   `OMI_PHYSICAL_QUALIFICATION=true` and `OMI_AUTH_GATEWAY=local_issuer`
   succeeds. A device run reaches the `fixture_signin` and `resolve_auth`
   stages against the fixture server.
8. The grep at the top of this table returns only the excluded paths, and
   check 1 of "Client: Firebase retirement" still returns nothing.

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

**Transport: raw WebSocket on Python 3.11.** The backend images are pinned to
Python 3.11 (`backend/Dockerfile:1`, `backend/pusher/Dockerfile:1`).

- **Chosen.** Call the `StartStreamTranscriptionWebSocket` endpoint directly.
  Build the presigned `wss://transcribestreaming.ap-southeast-2.amazonaws.com:8443`
  URL with botocore `SigV4QueryAuth`, connect with the already-pinned
  `websockets==12.0` (`backend/requirements.txt:236`), decode responses with
  botocore's `eventstream.EventStreamBuffer`, and encode audio frames with a
  small in-tree event-stream encoder (prelude, headers, CRC32 from `zlib`).
  `boto3` and `botocore` are in neither image's lock today. They are added to
  both, as described under "Python dependency closure".
- **Rejected: `aws-sdk-transcribe-streaming`.** PyPI lists it as a developer
  preview that requires Python 3.12 or later.
- **Rejected: `amazon-transcribe`.** Deprecated, and it depends on `awscrt`.
- **Rejected: upgrading the images to Python 3.12.** It re-resolves the whole
  dependency lock for one provider.
- `UNVERIFIED`: botocore's event-stream decoder is an internal module, not a
  documented public API. Pin the botocore version and cover the decoder with a
  framing test.

### STT: seam contract

The STT provider set is closed. `STT_SERVICE_MODELS=transcribe` is rejected
today because `STTService` (`utils/stt/streaming.py:122`) has no member for it,
`provider_for_model_token('transcribe')` (`config/stt_provider_policy.py:324`)
returns `None`, `PROVIDER_SERVING_SURFACES` (106) omits it, and
`get_stt_service_for_language` (`streaming.py:962`) has no branch for it. The
STT card adds a branch or set entry at each seam below.

| Seam | Change |
| --- | --- |
| `config/stt_provider_policy.py` provider constants (lines 25-29) | Add `TRANSCRIBE_PROVIDER = 'transcribe'` |
| `PROVIDER_SERVING_SURFACES` (106) | `TRANSCRIBE_PROVIDER: {STREAMING}`. Prerecorded and PTT stay off until phase 2 |
| `DEFAULT_MODELS_BY_SURFACE` (145) and `canonical_model_config` (385) | Streaming default `('transcribe',)`. `scripts/runtime_env_validation/manifest.py` checks the deployed `STT_SERVICE_MODELS` in `deploy/runtime_env/_base.yaml` against this, so the fork's runtime env file changes in the same card |
| `provider_for_model_token` (324), `provider_for_service` (390) | Map the `transcribe` token and service to the provider |
| Language policy in `stt_provider_policy.py` | A `TRANSCRIBE_SUPPORTED_LANGUAGES` set beside `MODULATE_SUPPORTED_LANGUAGES`, from the Transcribe streaming language table |
| `config/policy/listen.py` | `service_models` default |
| `utils/stt/streaming.py` `STTService` (122) | Add `transcribe`, with its model name and circuit-breaker key |
| `_primary_streaming_service` (313), `get_stt_service_for_language` (962), `connect_stt_socket_with_fallback` (457) | Selection and connect branches. With one provider there is no fallback hop |
| `validate_streaming_stt_env` (861) | New check: fail startup when `transcribe` is listed and no AWS region is configured |
| New `utils/stt/transcribe.py` | `TranscribeStreamingSocket(STTSocket)` and `process_audio_transcribe`, emitting the existing connect-latency and stream-close metrics |
| `utils/stt/live_session.py`, `utils/stt/live_chain.py`, `config/live_stt_registry.py` | Session build branch, callbacks, demand lane, and the `DEFAULT_TARGETS` and `Target.capable` entries |
| `routers/listen/receiver.py` legacy provider branch | Add the branch, or route `transcribe` through the managed chain only |
| Closed provider sets in the resilience, allowance-watch, connect-backoff, stream-close, live-health, live-metrics, provider-stream-end and vendor-guard modules | Add the provider, or the provider is treated as unknown |
| Pusher image dependencies | `pusher/main.py:30` imports `validate_streaming_stt_env` from the shared `utils/stt/streaming.py` and calls it at startup (55). Once that module imports `utils/stt/transcribe.py`, the pusher image needs `botocore` too. Add it to `backend/pusher/requirements.txt` and regenerate `backend/pusher/pylock.toml`. See "Python dependency closure" |
| Task IAM role | `transcribe:StartStreamTranscriptionWebSocket` |
| Tests | `tests/unit/test_stt_provider_policy.py` canonical-config cases, plus new framing and speaker-mapping tests for `transcribe.py` |

The repo-wide sweep for the closed sets is `grep -rn SONIOX_PROVIDER backend/`
and `grep -rn "STTService.soniox" backend/`. Every hit is a place the new
provider must appear or be deliberately absent.

**Runtime config:** `STT_SERVICE_MODELS=transcribe`, `SONIOX_API_KEY` and
`MODULATE_API_KEY` unset, `STT_GPU_FLEET_OFF` unset. `validate_streaming_stt_env`
also raises when `soniox` is listed without a key, or when `STT_GPU_FLEET_OFF`
is true without one.

**Accepted limitation.** A single streaming provider with no failover hop. A
Transcribe outage ends live capture until it recovers. Video attachments in chat
get an empty transcript, because prerecorded Transcribe is phase 2.

### Python dependency closure

The backend and pusher images install from separate locks. `backend/Dockerfile:22`
syncs `backend/pylock.runtime.toml`, which is compiled from
`backend/requirements.txt`. `backend/pusher/Dockerfile:20-21` syncs only
`backend/pusher/pylock.toml`, which is compiled from the smaller
`backend/pusher/requirements.txt`. The pusher image copies the shared
`config/`, `database/`, `routers/`, `services/` and `utils/` trees
(`pusher/Dockerfile:44-52`). Any third-party package that shared code
imports must therefore be in both requirement files. A package added only to
`backend/requirements.txt` fails at import time in pusher.

New packages and the image that reaches each one:

| Package | Reached from | `requirements.txt` | `pusher/requirements.txt` |
| --- | --- | --- | --- |
| `boto3`, `botocore` | S3 storage client, Cognito `AdminGetUser`, Transcribe socket and presigning, Secrets Manager reads | add | add. Pusher reaches the storage client during finalization, and `validate_streaming_stt_env` at startup |
| `langchain-aws` | `utils/llm/providers.py` `bedrock` provider and `BedrockEmbeddings` | add | add. Pusher runs `process_conversation`, which calls `get_llm` for the conversation and memory features |
| `anthropic[bedrock]` extra | `AsyncAnthropicBedrock` in `utils/retrieval/agentic.py` | change `anthropic>=0.52.0` to `anthropic[bedrock]>=0.52.0` | same change. The bare `anthropic` pin is already in both files |
| `PyJWT` | `verify_token` in `utils/other/endpoints.py` | already pinned (`requirements.txt:173`) | not added. Pusher does not import `endpoints.py`. The closure probe below confirms it |
| `certifi` | `database/redis_connection.py::redis_tls_kwargs` CA bundle (see "Valkey: TLS on every client") | already pinned (`requirements.txt:23`) | add `certifi==2026.10.18` |
| `pypdf` | PDF page count in `FileChatTool.upload` (see "Attach contract") | add | add. Pusher's image already carries `chat_file` through `tools/__init__` → `file_tools` (`pusher/requirements.txt:92`), so the import closure reaches it |

Every package uses the same pin in both files. `psycopg[binary]==3.2.10`,
`redis==5.0.8` and `websockets==12.0` are already in both. After editing both
requirement files, run `backend/scripts/update-python-lock.sh`. It regenerates
`pylock.toml`, `pylock.runtime.toml`, the macOS and Windows locks, and
`pusher/pylock.toml` (line 46) in one pass.

**Verification in each implementation card that adds an import:**

1. Build both images, `backend/Dockerfile` and `backend/pusher/Dockerfile`.
2. Run `python3 backend/scripts/runtime_image_contracts.py smoke --dockerfile backend/pusher/Dockerfile --image <pusher image>`,
   and the same with `backend/Dockerfile` and the backend image. `smoke_image`
   (`runtime_image_contracts.py:812`) imports every third-party module the
   image's first-party closure reaches, then imports the entrypoint. It fails
   when a lock is missing a package.
3. Extend `tests/unit/test_runtime_image_contracts.py` the way
   `test_pusher_dependency_probe_includes_jsonschema` (361) does: assert that
   `botocore` and `langchain_aws` are in the pusher contract's
   `third_party_dependency_modules`.

### LLM: Bedrock through `langchain-aws`

- **Provider.** Add a `bedrock` provider beside `OPENAI_COMPATIBLE_PROVIDERS`
  (`utils/llm/providers.py:71`) that returns `ChatBedrockConverse`. Today every
  managed feature resolves to `('gpt-6-luna', 'openai')` in
  `utils/llm/model_config.py`. Add an `aws` profile to `MODEL_QOS_PROFILES` and
  set `MODEL_QOS=aws`. The profile covers every feature key in
  `model_config.py`, including `episode_composer` (line 113), which the
  episodes read path reaches through `utils/episodes/home_read.py:389`
  `schedule_missed_composes`.
- **Models** (verified in ap-southeast-2 with `list-inference-profiles`):
  `au.anthropic.claude-sonnet-4-5-20250929-v1:0` for `chat_agent` and
  `conv_structure`, and `au.anthropic.claude-haiku-4-5-20251001-v1:0` for the
  other features. The `au.` profiles keep inference in Australian regions. The
  task role allows only `au.*` profiles and denies `global.*`.
- **Anthropic Messages client.** `_AnthropicClientProxy`
  (`utils/llm/clients.py:276`, used by `agentic.py:963`) switches its default
  client to `anthropic.AsyncAnthropicBedrock`. It reaches the gateway only when
  `should_route_chat_agent_through_gateway()` is true (`clients.py:298`), which
  needs `OMI_LLM_GATEWAY_FEATURE_MODE=gateway`. That variable stays unset.

#### Managed dispatch: Bedrock before the gateway branch

**Today every managed call goes to the gateway, whatever the env says.**
`get_llm` (`clients.py:762`) computes `route_through_gateway` at 883 from
`should_route_company_paid_features_through_gateway()`, which returns `True`
unconditionally (`gateway_client.py:231-239`). The optional
`OMI_LLM_GATEWAY_FEATURE_MODE` and `OMI_LLM_CHAT_AGENT_ROUTE` settings are
read only for BYOK (`clients.py:871-880`). A keyless request therefore takes the
managed branch at 915-923, `get_or_create_omi_gateway_llm(feature_auto_lane_id(feature))`,
before the direct provider factory at 925-930 is considered. The gateway base
URL defaults to `http://127.0.0.1:9080` (`gateway_client.py:30`) when
`OMI_LLM_GATEWAY_URL` is unset. Neither the `backend` nor the `pusher`
container in the task listens there, so adding an `aws` profile alone leaves
chat, conversation structure and episode compose failing with a connection
error. Leaving `OMI_LLM_GATEWAY_FEATURE_MODE` unset changes nothing on that
path.

**Decision.** `get_llm` dispatches the `bedrock` provider explicitly, right
after the Anthropic and Perplexity guards (`clients.py:786-793`) and before
BYOK resolution (810) and the gateway branch:

```python
model, provider = _get_model_config(feature)
...  # existing anthropic / perplexity guards
if provider == 'bedrock':
    _require_client_budget(feature, uid=uid, provider='bedrock', allow_byok=False)
    result = get_or_create_bedrock_llm(
        model, streaming, {'request_timeout': request_timeout, 'max_retries': max_retries}
    )
    return result.model_copy(update={'metadata': {**(result.metadata or {}), 'omi_resolved_model': model}})
```

- `get_or_create_bedrock_llm` lives in `utils/llm/providers.py` beside
  `get_or_create_openai_compatible_llm` (101). It caches a `ChatBedrockConverse`
  per (model id, streaming, timeout, retries), with
  `region_name='ap-southeast-2'` and a `botocore.config.Config` whose
  `read_timeout` is the feature deadline and whose `retries` is
  `{'max_attempts': max_retries + 1, 'mode': 'standard'}`.
- Only the `aws` profile maps a feature to `bedrock`. Every upstream profile
  keeps its `(model, provider)` pairs, so upstream dispatch, the BYOK branch
  and the gateway invariant are byte-for-byte unchanged when `MODEL_QOS` is
  not `aws`. `should_route_company_paid_features_through_gateway` keeps
  returning `True`.
- BYOK is not consulted for a `bedrock` feature. The fork has no BYOK users,
  and a client-supplied key must not switch an AWS-profile request onto a
  provider the task cannot reach.
- The prompt-cache binds at the end of `get_llm` (`clients.py:938-948`) are
  skipped by the early return. They are keyed on OpenAI model names
  (`supports_prompt_cache`, `uses_explicit_cache_and_chat_sanitizer`,
  `model_config.py:410-423`) and would not match an `au.anthropic.*` id
  anyway.
- `MODEL_QOS` falls back to `premium` with only a warning when the name is not
  a profile (`model_config.py:182-184`), which would silently put every feature
  back on the gateway. The port adds a startup check: when `MODEL_QOS=aws` is
  set and `get_active_profile_name()` is not `aws`, `main.py` and the pusher
  entrypoint raise.

**Rejected: run `backend/llm_gateway` as a third container with a Bedrock
upstream adapter.** It keeps the gateway invariant intact, but it adds a
service, a service token in Secrets Manager and a second hop to every call,
and the gateway's lanes are OpenAI chat-completions shaped, so Bedrock would
still need an adapter there. At one user there is no spend ledger to protect.

**Rejected: make `should_route_company_paid_features_through_gateway` return
`False` under the `aws` profile.** That sends managed calls to
`get_default_client` (`providers.py:254`), which knows only Gemini and the
OpenAI-compatible providers, and it weakens the upstream rule that no unset
flag can send paid traffic direct.

#### Direct gateway callsites

`get_llm` is not the only way onto the gateway. Every backend callsite of a
gateway client or lane (`git grep -n "get_or_create_omi_gateway_llm\|get_llm_gateway_client\|get_llm_gateway_base_url\|invoke_.*_gateway\|run_proactivity_gateway\|get_file_chat_gateway" backend -- ':!backend/tests' ':!backend/llm_gateway' ':!backend/testing' ':!backend/evals'`)
is ported or cut below:

| Callsite | Reachable in the MVP | Disposition |
| --- | --- | --- |
| `clients.py:915-923` managed `get_llm` branch | yes: every MVP feature | Ported: the `bedrock` dispatch above returns before it |
| `clients.py:889-905` BYOK gateway branch | no: the `bedrock` dispatch returns before BYOK resolution | Unreachable under `aws` |
| `clients.py:952` `get_llm_gateway_chat_structured` | no: imported by `conversation_processing.py:36` but called nowhere | Unreachable |
| `utils/memory/required_intake.py:56-63` `normalize`, the canonical memory required-intake Flex call (`memory_l2_flex`) | yes: `MEMORY_REQUIRED_INTAKE_ENABLED` defaults on (`config/required_memory_intake.py:17-18`). `POST /v3/memories` writes through `MemoryService._canonical_write` (`memory_service.py:3955-3958`), which marks the item required with `required_processing_payload`, then `write_canonical_external_memory` (`canonical_memory_adapter.py:1933`) reaches `write_canonical_extraction_memory`, which calls `after_capture` (1828) inside its 12 s inline budget. Conversation memory extraction writes through the same function | Ported, see "Flex lanes" below |
| `utils/memory/promotion_flex.py:41-47` `_get_gateway_flex_llm` and its `.bind(service_tier='flex')` (239) | no: `read_promotion_flex_control` returns the standard control unless `OMI_BACKGROUND_FLEX_CAPABLE` is true (`promotion_flex.py:87-91`, default `false`), and `llm_for_uid` then returns `None` (169-170) | Cut: `OMI_BACKGROUND_FLEX_CAPABLE` stays unset. The router's users (`utils/llm/memories.py:430`, `canonical_consolidation.py:899`, `x_connector.py:41`) keep their standard `get_llm` model, which takes the `bedrock` dispatch |
| `utils/task_intelligence/workstream_association.py:78` `_default_adjudicator` → `invoke_chat_structured_gateway` (`gateway_client.py:291`) | yes: `process_conversation.py:1945` calls it after memory extraction whenever the owner has an open workstream candidate | Ported: `invoke_chat_structured_gateway` gains the same `bedrock` check. Under the `aws` profile it returns `get_llm('chat_extraction', request_timeout=timeout_seconds, max_retries=0).with_structured_output(output_model).invoke(prompt)` and keeps returning `None` on any model error, so `_default_adjudicator` still answers `model_unavailable` (85) |
| `utils/other/chat_file.py:419` and `:475`, the `get_file_chat_gateway_async_client` / `_sync_client` file-chat lanes | yes: chat with an attachment | Ported by "Chat files: S3 plus Bedrock content blocks", which replaces both `_ask_files*` paths |
| `utils/retrieval/tools/perplexity_tools.py:150` `_post_gateway_chat_completion` | yes today: `agentic.py:1694` registers the tool unconditionally | Cut: `OMI_WEB_SEARCH_TOOL=off` (see "Cut switches") |
| `routers/public_shared_conversation_chat.py:314` | no: the web share page, not the Flutter app | Cut: `OMI_DISABLED_ROUTERS=public_shared_conversation_chat` |
| `routers/desktop_chat.py:1584`, `:1903` (`POST /v2/chat/completions`) | no: the macOS app only | Cut: `OMI_DISABLED_ROUTERS=desktop_chat` |
| `utils/llm/app_generator.py:191` `generate_image_via_gateway` | no: `routers/apps.py` only | Cut with the `apps` router |
| `utils/proactivity.py:82` `run_proactivity_model` → `run_proactivity_gateway` | no: `run_proactivity_model` has no caller at the base commit | Unreachable |
| `utils/llm/clients.py:441-476` OpenAI embeddings and `:1100-1102` Gemini embeddings gateway | no: `EMBEDDINGS_PROVIDER=bedrock` and the cut index (see "Embeddings") | Unreachable |
| `utils/llm/gateway_anthropic.py`, `utils/llm/desktop_gemini_gateway.py`, `utils/dream_capacity.py:90`, `utils/dream_harness_transport.py:277` | no: chat-agent gateway route off, desktop proxy, and Dream (routers cut, `DREAM_AGENT_ENABLED` off) | Unreachable |

**Flex lanes.** `required_intake.normalize` asks the gateway for its
`memory_l2_flex` lane and binds `service_tier='flex'` itself. Bedrock has no
OpenAI Flex lane, and an unreachable gateway would defer every required memory
to Dream retry, which the MVP cuts. A new `utils/llm/clients.py::get_flex_llm(feature, *, request_timeout)`
owns the choice for both Flex callers:

- When `_get_model_config(feature)` names `bedrock`, it returns
  `get_llm(feature, request_timeout=request_timeout, max_retries=0)` with no
  `service_tier` bind. The `aws` profile maps `memory_l2_flex` to Haiku.
- Otherwise it returns today's `get_or_create_omi_gateway_llm(...).bind(service_tier='flex')`.

`required_intake.normalize` and `promotion_flex._get_gateway_flex_llm` call
`get_flex_llm` and drop their own `.bind(service_tier=...)`, so upstream sends
the same request as before. The required-intake error classification
(`required_intake.py:68-76`) also maps botocore's `ReadTimeoutError`,
`ConnectTimeoutError` and a `ClientError` with code `ThrottlingException`,
`ServiceUnavailableException` or `ModelNotReadyException` to
`RequiredProcessingDeferred`, so a Bedrock outage does not spend the
processor's quality retry budget.

#### Request options at bind time

The provider constructor is not the only place OpenAI arguments enter a
request. Callers bind request kwargs after `get_llm` returns. For example, the
chat runner binds `max_completion_tokens=8192` (`agentic.py:1230`).
`ChatBedrockConverse._stream` camel-cases every bound kwarg and passes it to
`_converse_params`
([`bedrock_converse.py:1502-1518`](https://github.com/langchain-ai/langchain-aws/blob/214673ad41997525d12c2c5427431eb0626abb39/libs/aws/langchain_aws/chat_models/bedrock_converse.py#L1502-L1518)),
whose keyword-only signature accepts `maxTokens`, `tools`, `toolChoice`,
`serviceTier` and a few others, but not `maxCompletionTokens`, `extraBody` or
`promptCacheKey`
([`2048-2071`](https://github.com/langchain-ai/langchain-aws/blob/214673ad41997525d12c2c5427431eb0626abb39/libs/aws/langchain_aws/chat_models/bedrock_converse.py#L2048-L2071)).
An unknown key raises `TypeError` before any Converse request. `reasoning_effort`
is popped and validated against the model profile's `reasoning_effort_levels`
(1265-1290). It is ignored with a warning when the profile declares none, and
raises `ValueError` when the effort is not one of the declared levels.

**Decision.** Provider-specific request options go through one module,
`utils/llm/request_options.py`, and no retained path binds an OpenAI-only
kwarg directly. It unwraps a `RunnableBinding` with `getattr(model, 'bound', model)`,
as `notes_effort.py:10` already does, and branches on
`isinstance(underlying, ChatBedrockConverse)`:

| Helper | OpenAI-compatible model (unchanged upstream request) | `ChatBedrockConverse` |
| --- | --- | --- |
| `bind_chat_tools(model, tools, *, tool_choice='auto', max_output_tokens, extra_body=None)` | `model.bind(tools=tools, tool_choice=tool_choice, max_completion_tokens=max_output_tokens, **({'extra_body': extra_body} if extra_body else {}))` | `model.bind_tools(tools, tool_choice=tool_choice, max_tokens=max_output_tokens)`. `bind_tools` (1704-1787) converts the OpenAI tool dicts and resolves `'auto'` to `{'auto': {}}`. `max_tokens` becomes `inferenceConfig.maxTokens`. `extra_body` is dropped |
| `bind_reasoning_effort(model, effort)` | `model.bind(reasoning_effort=effort)` | returns `model` unchanged and records `record_fallback(component='llm_request', from_mode='reasoning_effort', to_mode='model_default', reason='unsupported', outcome='degraded')` |
| `bind_service_tier(model, tier)` | `model.bind(service_tier=tier)` | returns `model` unchanged |
| `bind_openai_cache(model, **params)` (`extra_body`, `prompt_cache_key`) | `model.bind(**params)` | returns `model` unchanged |

**Bind and invoke kwargs on the retained LLM paths** (`git grep -n "\.bind(" backend/utils backend/routers`, plus every `model_copy(update=...)` on a chat model, filtered to paths the MVP reaches):

| Site | Kwargs today | Under the `aws` profile |
| --- | --- | --- |
| `utils/retrieval/agentic.py:1230` `_run_openai_agent_stream` | `tools`, `tool_choice='auto'`, `max_completion_tokens=8192` | `bind_chat_tools(..., max_output_tokens=8192)` |
| `utils/retrieval/shaped_chat.py:81-84` `run_shaped_chat_stream` (the chat runner when `OMI_SHAPED_AGENT_MODE=on`, see "Runtime settings") | `tools`, `tool_choice='auto'`, `max_completion_tokens=8192`, and `extra_body` when the GPT-5.6 cache is on | `bind_chat_tools(..., max_output_tokens=8192, extra_body=...)`. `cache_enabled` is already false, because `model_supports_explicit_cache` (`prompt_cache.py:27`) does not match a Bedrock id |
| `utils/llm/notes_effort.py:31` `bind_notes_effort`, from `conversation_processing.py:570` (`conv_structure` on escalation) | `reasoning_effort='xhigh'` | Its model-name guard (10-14) already returns the model unchanged, because `ChatBedrockConverse.model` (a property over `model_id`, `bedrock_converse.py:754`) is an `au.anthropic.*` id that fails the GPT prefix check. It moves onto `bind_reasoning_effort` so the provider decision has one owner instead of a name match |
| `utils/llm/shaped_notes_transport.py:71-72` `isolated_notes_model`, around every `conv_structure` notes call (`conversation_processing.py:567`) and `notes_agent_serving.py:180` | n/a. It raises `TypeError` unless the model is `ChatOpenAI` or `ChatAnthropic` | Adds a `ChatBedrockConverse` branch that yields the model unchanged. `ChatBedrockConverse` defines no async methods, so `ainvoke` runs the sync boto3 call in LangChain's executor, and no connection pool is bound to the notes worker's event loop |
| `utils/llm/wake_word_adjudication.py:163` | `reasoning_effort='high'`, bound because the feature-mode flag is unset | `bind_reasoning_effort` |
| `utils/memory/required_intake.py:63`, `promotion_flex.py:239` | `service_tier='flex'` | `get_flex_llm` ("Flex lanes") |
| `utils/llm/prompt_cache.py:139` `with_cache_write_opt_out`, from `utils/llm/chat.py:1235` and `:1342` (metadata extraction in chat) | `extra_body={'prompt_cache_options': ...}`, unless BYOK | `bind_openai_cache`. Today it binds for every managed model |
| `utils/llm/prompt_cache.py:155` `bind_explicit_cache`, from `proactive_notification.py:452` | `extra_body`, `prompt_cache_key` | `bind_openai_cache` |
| `utils/llm/clients.py:938-948`, `:976` | `prompt_cache_key`, `extra_body` | Skipped by the early `bedrock` return, and keyed on OpenAI model names |
| `utils/translation_core/providers.py:102-108` `LunaTranslationProvider.translate` (`viewed_v1` policy) | `model_copy(update={'max_tokens': ..., 'max_output_tokens': ...})`, after `get_model('translation') != self.model_name` (98), where `model_name = LUNA_MODEL` (82) | The identity check compares against `get_model('translation')` resolved at construction, so a Bedrock model is not rejected as `config_incomplete`. The update sets only `max_tokens`, a declared `ChatBedrockConverse` field |
| `utils/episodes/composer.py:697` | `tools` only | Unchanged. `tools` is a `_converse_params` key, and `_format_tools` (3824) converts OpenAI tool dicts |
| `utils/memory_ingestion/adapters/production_like_model.py:788` | `temperature` | Unchanged. `temperature` is accepted |

**Message shapes.** The chat runners keep OpenAI message dicts in history: an
assistant turn with `tool_calls[].function.arguments` as a JSON string, and
`{'role': 'tool', 'tool_call_id': ..., 'content': ...}` results
(`agentic.py:1347-1363`, `shaped_chat.py:105-117`). LangChain's message
conversion turns those into `AIMessage.tool_calls` and `ToolMessage`, which
`_messages_to_bedrock` emits as `toolUse` and `toolResult` blocks. A system
message after the first turn (`chat_mount.py:62`) is hoisted into Converse's
`system` (2668-2685). The runners read text with `openai_content_text`
(`agentic.py:1118`), which already accepts the list-of-blocks content that
Converse chunks carry, and tool calls with `openai_tool_calls` (1158), which
reads `tool_call_chunks`.

**Tests (implementation card).** Every Bedrock test injects a fake
`bedrock-runtime` client through `ChatBedrockConverse(client=...)` that
records each `converse` / `converse_stream` call's keyword arguments and
returns a canned event stream. The fake runs against the `langchain-aws`
version pinned in both requirement files, so a library change that moves a
kwarg breaks the test.

- `tests/unit/test_llm_aws_dispatch.py`, with `MODEL_QOS=aws` and
  `OMI_LLM_GATEWAY_URL` unset: for `chat_agent`, `conv_structure`,
  `episode_composer`, `translation`, `chat_extraction` and `memory_l2_flex`,
  invoking `get_llm(feature)` sends exactly one request to the fake Bedrock
  client with the profile's `au.` model id, and the gateway factories
  (`get_or_create_omi_gateway_llm`, `get_llm_gateway_client`) are patched to
  raise, so a call that reaches them fails the test. A second case with
  `MODEL_QOS` unset asserts the managed branch still calls
  `get_or_create_omi_gateway_llm` (upstream unchanged). A third sets a BYOK
  OpenAI key under `aws` and asserts the request still reaches Bedrock.
- Agent loop, for both `_run_openai_agent_stream` and `run_shaped_chat_stream`:
  the fake's first `converse_stream` emits a `toolUse` start, two
  `toolUse` input deltas and `stopReason=tool_use`. The test asserts the
  first call's `inferenceConfig.maxTokens == 8192` and
  `toolConfig.toolChoice == {'auto': {}}`, with every chat tool in
  `toolConfig.tools`, that the runner executed the named tool once with the
  joined arguments, and that the second call's `messages` holds an assistant
  `toolUse` block and a user `toolResult` block with the same `toolUseId`.
  The second call returns text, and the test asserts it reaches
  `full_response`.
- `invoke_chat_structured_gateway` under `aws` sends one Bedrock request and
  no gateway request, and returns `None` when the fake raises.
- `required_intake.normalize` under `aws` sends one Bedrock request with no
  `serviceTier`. A botocore `ReadTimeoutError` from the fake raises
  `RequiredProcessingDeferred`. Upstream, `get_flex_llm` still binds
  `service_tier='flex'` on the gateway model.
- `request_options`: each helper's OpenAI branch binds exactly the kwargs in
  the table, and each `ChatBedrockConverse` branch produces a `converse`
  request with no `TypeError`. A census test walks the call graph and asserts
  no retained path calls `.bind(max_completion_tokens=`, `.bind(extra_body=`,
  `.bind(reasoning_effort=` or `.bind(service_tier=` outside
  `request_options.py`.
- `isolated_notes_model` yields a `ChatBedrockConverse` unchanged, and one
  `get_conversation_notes` call under `aws` reaches the fake.

### Embeddings: Titan input bound and the cut index

**No embedding call while the index is cut.** Pinecone is unconfigured in the
MVP, so `database/vector_db.index` is `None` and every vector write or query
is discarded. Several callers still embed first and learn that later:
`save_structured_vector` (`process_conversation.py:2191`) calls
`generate_embedding` and, for external-integration conversations, an LLM
metadata extraction (2204-2210), and only then reaches `upsert_vector2`, which
returns `False` on `index is None` (`vector_db.py:173`). That runs once per
completed conversation from the finalizer (`finalizer.py:111`), from
`process_conversation.py:3107`, and from `smart_merge.py:996`.

The port adds one predicate, `vector_db.vector_index_configured() -> bool`
(`index is not None`), and every caller that pays for a provider call before
reaching the index checks it first:

| Caller | Today | Port |
| --- | --- | --- |
| `save_structured_vector` (2175) | embeds, extracts metadata, then `upsert_vector2` returns `False` | first check after the existing skip guards: when unconfigured, count `outcome='skipped_index_unconfigured'` and return `True` with no provider call. `True` because nothing is pending, so `apply_settled_enrichment` (2107) does not raise "vector is not ready" and the finalizer does not record a degraded fallback for a deliberate cut |
| `upsert_vector2` (172) | counts `outcome='error'` on `index is None` | counts `skipped_index_unconfigured`, still returns `False` for any direct caller |
| `query_vectors` and the other `vector_db` helpers | check `index is None` before embedding (222, 356, 469, 564, 781 and the rest) | unchanged |
| `utils/mcp_server/handlers/conversations.py:454` `search_conversations_core` | passes `embed_query`, so `resolve_mcp_conversation_search_ids` embeds before `query_vectors` returns empty | passes `embed_query=None` and skips the call when unconfigured, then falls through to its existing keyword search (487-497) |
| `utils/app_integrations.py:446` `_retrieve_contextual_memories` | embeds the question, then `query_vectors_by_metadata` returns empty | returns `[]` before embedding when unconfigured. Its caller is app proactive notifications, cut with `apps` |
| `routers/conversations.py:2481`, `utils/retrieval/tool_services/conversations.py:330`, `utils/retrieval/tools/conversation_tools.py:786` | already gate the embed on `index is not None` / `index_available` | unchanged |

`OMI_CONVERSATION_SUMMARY_VECTOR_UPSERTS_TOTAL` (`utils/metrics.py:426`)
gains the `skipped_index_unconfigured` label value, so a cut index reads as
skipped rather than as an error rate of 100%.

**Titan input bound.** Titan Text Embeddings V2 accepts at most 8,192 tokens
or 50,000 characters per input and rejects a longer input with a 400
`ValidationException` ("Too many input tokens") instead of truncating it
([model card](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html)).
AWS publishes only an average English ratio of about 4.7 characters per
token. No character count guarantees a token count: code, CJK text and emoji
can run below one character per token. Titan has no token-counting API (the
Bedrock `CountTokens` model cards list no Titan embedding model), and the existing fitter
(`_fit_openai_embedding_text`, `gateway_client.py:726`) counts with OpenAI's
`cl100k` encoding and runs only on the gateway path, which this port leaves
off. So the bound is enforced by Titan itself, with a deterministic retry:

1. `_fit_titan_input(text)` removes NUL bytes, maps an empty result to one
   space (as the OpenAI fitter does), and keeps at most the first 50,000
   characters, which is Titan's hard character limit.
2. `_TitanEmbeddings.embed_one(text)` calls `invoke_model` with that input.
3. On a `ValidationException` whose message contains "too many input tokens"
   (compared case-insensitively), it halves the input (keeps the first
   `len // 2` characters, cut back to the last whitespace in the final 10% when
   there is one) and calls again. It stops after 5 halvings. The last attempt
   is at most 1,562 characters, which stays under 8,192 tokens even at 4
   tokens per character (a byte-level tokenizer's worst case, one token per
   UTF-8 byte). A sixth rejection raises `EmbeddingInputTooLong`, which the
   callers already treat as a provider failure (the finalizer absorbs it,
   `finalizer.py:102-130`).
4. Any other `ValidationException`, and every other error, is raised unchanged
   on the first attempt. Throttling keeps botocore's `standard` retry mode.

Prefix truncation matches the current gateway fitter's behaviour: the vector
represents the start of the text. `embed_documents` applies `embed_one` to each
input in order, so positional alignment with the caller's list holds. Each
retry logs `embedding_fit outcome=halved chars=<n>` and counts
`omi_embedding_input_fit_total{outcome}` (`fit`, `halved`, `too_long`), so a
workload that keeps hitting the retry is visible before Pinecone or pgvector is
turned on.

**Rejected: a fixed character cap such as 24,000 characters.** A cap that is
safe for every script would be the 1,562-character floor above, which would
discard most of a long English summary on every call. A larger cap is a guess
that fails for dense text.
**Rejected: a local tokenizer.** AWS does not publish Titan's tokenizer, and
`cl100k` is a different vocabulary, so it would give a confident wrong count.

**Seam.** `utils/llm/clients.py` builds `_TitanEmbeddings` (a thin
`BedrockEmbeddings` subclass that overrides `embed_query`, `embed_documents`
and their async forms) as `_OpenAIEmbeddingsProxy._default` when the new
`EMBEDDINGS_PROVIDER=bedrock` setting is present. The BYOK branch keeps OpenAI. `langchain-aws`
re-raises the botocore `ClientError` unchanged in current releases; versions
before that wrapped it in `ValueError("Error raised by inference endpoint: …")`.
The retry calls `self.client.invoke_model` directly, so the pinned
`langchain-aws` version cannot change what it sees.

**Tests (embeddings card).**

- With a botocore `Stubber`: an input rejected once with "Too many input
  tokens" is retried at half length and returns the second vector. Five
  rejections then success returns a vector from a ≤ 1,562-character input.
  Six rejections raise `EmbeddingInputTooLong`. A `ValidationException` with
  another message, and an `AccessDeniedException`, are raised after one call.
- 60,000 characters are sent as 50,000. `""` and `"\x00"` are sent as `" "`.
- `embed_documents([a, b, c])` returns three vectors in order when `b` needed
  a retry.
- With `vector_db.index = None`: `save_structured_vector` returns `True`,
  counts `skipped_index_unconfigured`, and the stubbed embeddings and
  `retrieve_metadata_from_*` record zero calls. `apply_settled_enrichment`
  does not raise. `search_conversations_core` makes no embedding call and
  returns its keyword-search result.
- Deployed, with Pinecone unset: finalize one conversation and confirm that
  CloudTrail shows no `InvokeModel` on `amazon.titan-embed-text-v2:0` for it.

### Chat files: S3 plus Bedrock content blocks

`POST /v2/files` and `/v2/files/video` upload attachments to OpenAI Files today
(`utils/other/chat_file.py:262` and 343), and the answer path calls the LLM
gateway's file-chat lanes (`_ask_files_stream` at 397 through
`get_file_chat_gateway_async_client`, 419; `_ask_files` at 462 through
`get_file_chat_gateway_sync_client`, 475).

- Store the bytes in `BUCKET_CHAT_FILES` under `{uid}/files/{file_id}`. Keep
  the `openai_file_id` field on `u/files/{id}` and put the S3 key in it, so the
  rows and `download_owned_chat_file` (115) keep their shape. A video's
  filmstrip goes to `{uid}/files/{file_id}.strip.jpg`, and that key replaces
  the OpenAI id in `video.strip_file_id`.
- At answer time, read the object and send it to `ChatBedrockConverse` as an
  image or document content block, replacing both `_ask_files*` paths. What
  the upload may store, and how many blocks one request may carry, follow the
  attach contract below.
- Thumbnails and deletion follow the private-object contract below.

#### Attach contract: admit only what Converse accepts

The current upload admits what OpenAI Files accepts, and the answer path
finds out later. `/v2/files` takes parts up to 50 MiB
(`utils/multipart.py:19` `CHAT_FILE_MAX_PART_SIZE`, `routers/chat.py:1519`).
Its document test is `chat_file_is_document` over
`CHAT_FILE_DOCUMENT_EXTENSIONS` (`models/chat.py:69-206`), and its image test
is any `image/*` MIME that Pillow can open (`chat_file.py:230`, `:270-283`).
Converse is narrower
([DocumentBlock](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_DocumentBlock.html),
[ImageBlock](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_ImageBlock.html),
[Message](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_Message.html),
[inference restrictions](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-api-restrictions.html)):

- Document `format` is one of `pdf`, `csv`, `doc`, `docx`, `xls`, `xlsx`,
  `html`, `txt`, `md`. Each document is at most 4.5 MB, and a message carries
  at most 5. The document `name` allows only letters, digits, single spaces,
  hyphens, parentheses and square brackets, and AWS advises a neutral name
  because the model reads it.
- Image `format` is one of `png`, `jpeg`, `gif`, `webp`. Each image is at
  most 3.75 MB and 8000 × 8000 px, and a message carries at most 20. Claude
  reads only the first frame of an animated GIF.
- A message that carries a document also carries a text block.
- The whole request stays under the 25 MB invocation quota.

Uploads that pass today and would fail at answer time include every PPTX the
app's picker offers (`app/lib/providers/message_provider.dart:487-508`), PPT,
RTF, ODT, every code suffix, any document over 4.5 MB, and BMP or TIFF photos.
The port moves every one of those checks to upload time, so a file that is
stored can always be sent.

**Admission table.** `FileChatTool.upload` classifies each file once, by
suffix, or by MIME when the name has no suffix, exactly as
`chat_file_is_document` does today. It records the result in a new
`bedrock_format` field on `FileChat`. The answer path reads that field and
never re-derives the format.

| Class | Inputs | Stored object | `bedrock_format` |
| --- | --- | --- | --- |
| Native document | `pdf`, `csv`, `doc`, `docx`, `xls`, `xlsx`, `html`/`htm`, `txt`/`text`, `md`/`markdown` | original bytes | the matching Converse format (`htm` → `html`, `text` → `txt`, `markdown` → `md`) |
| Text document | every other suffix in the "Text and code" group of `CHAT_FILE_DOCUMENT_EXTENSIONS` (code, config, `json`, `xml`, `yaml`, `log`, `srt`, `vtt`, `ics`, `vcf`, `eml`, `mht`/`mhtml`, `mime`, `nws` and the rest), plus `tsv` and `iif` | original bytes, after a check that they decode as strict UTF-8 (BOM allowed) and contain no NUL byte | `txt` |
| Converted document | `pptx`, or MIME `application/vnd.openxmlformats-officedocument.presentationml.presentation` | Markdown text extracted from the slides (below) | `md` |
| Native image | Pillow `img.format` of `JPEG`, `PNG`, `GIF` or `WEBP` | original bytes, unless the size rule below re-encodes them | `jpeg`, `png`, `gif`, `webp` |
| Converted image | any other format Pillow opens (`BMP`, `TIFF`, `ICO`, `MPO`, `PPM` and the rest) | first frame re-encoded: PNG when it has an alpha channel, otherwise JPEG at quality 90 | `png` or `jpeg` |
| Rejected | `ppt`, `pot`, `pps`, `ppa`, `pwz`, `wiz`, `dot`, `odt`, `rtf`, `xla`, `xlb`, `xlc`, `xlm`, `xlt`, `xlw`; the Apple iWork and Google Apps MIME types; a text document that is not UTF-8; an image Pillow cannot open (HEIC, as today); everything else | nothing | n/a. 400 |

The image class goes by what Pillow decodes, not by the file name. A
converted image's row `mime_type` becomes `image/png` or `image/jpeg`, so the
client's `mimeTypeToFileType` (`schema/message.dart:124`), the thumbnail and
the answer path all agree on one format. Videos keep their route. The sheet
and filmstrip are JPEGs `video_frames.py` builds (383, 396).

**Size limits after classification:**

- **Upload part.** Stays at 50 MiB, which video clips need. It bounds the
  bytes read before classification, not what is stored.
- **Documents.** The stored object is at most 4,500,000 bytes, otherwise 400.
  A PDF must also open with `pypdf` and have at most 100 pages, otherwise 400.
  The count goes into a new `page_count` field. `UNVERIFIED`: whether the
  100-page Claude limit and the 4.5 MB cap apply to PDFs sent without
  citations through Converse. AWS says Claude 4 lifts the 4.5 MB cap for
  PDFs, but the console still enforces it, so the port keeps both limits.
- **PPTX extraction.** Read the archive with `zipfile`: at most 1,000
  members, each `ppt/slides/slide*.xml` part at most 10 MB decompressed and
  50 MB in total, and any part containing `<!DOCTYPE` is refused. Parse the
  slides in numeric order with `xml.etree.ElementTree`. Write `## Slide N`
  followed by the slide's `a:t` runs, one paragraph per `a:p`. Speaker notes
  and images are dropped. An archive that breaks a bound, or yields no text,
  gets a 400. The Markdown must fit the 4.5 MB document cap.
- **Images.** The stored object is at most 3,750,000 bytes and 8000 px on
  each side. A larger image is downscaled once, deterministically, to a
  2048 px longest edge (`Image.thumbnail`, `LANCZOS`). It is re-encoded in
  its own format: JPEG and WebP at quality 85, PNG with `optimize=True`,
  and GIF as its first frame in PNG. The row's `mime_type` and
  `bedrock_format` follow the stored bytes, so an oversized GIF becomes
  `image/png` and `png`. A result still over the cap gets a 400.
  The video sheet, the filmstrip and `window_sheet` output go through the
  same check before they are stored or sent.

**User-visible failure.** Every rejection raises `UnsupportedChatFileError`
with a reason the user can act on. For example: "'.pptx' files over 4.5 MB
of text can't be read in chat", "PDFs over 100 pages aren't supported in
chat", or "'.rtf' files aren't supported in chat. Save it as PDF or DOCX."
The routes already turn that error into a 400 with the text as `detail`
(`routers/chat.py:1544-1545`, `:1619-1620`). The client drops it today:
`_uploadFilesBatch` (`backend/http/api/messages.dart:445-458`) throws a bare
exception, and `uploadFiles` (`message_provider.dart:589-594`) shows the
generic "Failed to upload file". In the port `_uploadFilesBatch` throws a
`ChatFileRejectedException(detail)` on a 400 that carries a `detail`, and
`uploadFiles` shows that detail in the snackbar. Other failures keep the
generic text. The picker's `allowedExtensions` list is unchanged, because
every suffix in it is in a retained class.

**Answer-time request budget.** `get_chat_files_desc` returns up to 10 files,
newest first (`_requested_file_limit`, `chat_file.py:244`). The new
`_bedrock_blocks(question, files)` keeps that order. It takes:

- the question as the first text block;
- at most 5 documents, and PDFs only while their `page_count` total stays at
  or below 100;
- every image, never more than 10 from this list;
- only while the inline bytes stay at or below 20 MB.

Each included file has a text block naming it, followed by its content
block. The document block's `name` is the neutral `document-{n}`, so user
file names never reach the `name` field. Files that do not fit are listed in
a final text block ("Not included in this request: report.pdf (document
limit)"), so the model can ask the user to narrow the question. A row with
no `bedrock_format` gets the existing `UnsupportedChatFileError` text and
no provider call. No such row exists in the new store.

**Bounded S3 reads.** Every attachment, strip and thumbnail read is a
`GetObject` with `Range: bytes=0-{cap}`, where `cap` is the row's class
limit (4,500,000 or 3,750,000). It refuses a response whose `ContentRange`
total exceeds the cap. `download_owned_chat_file(max_bytes=...)` keeps its
contract with the same read. A missing key (`NoSuchKey`) raises
`StaleChatFileError`. A Converse `ValidationException` raises
`ProviderRejectedChatFileError`. Because admission already enforces each
file's format and size, `search_files_tool` treats that exception as a
request-shape fault (`file_tools.py:198-211`): it logs at error level and
returns its existing "I encountered an error while searching the files"
reply. `_provider_400_names_request_field` reads OpenAI error fields and
gets a Bedrock branch that always answers true.

**Rejected: keep the OpenAI allowlist and let Converse reject at answer
time.** The upload succeeds and shows a thumbnail. The failure arrives later,
as a generic chat error the user cannot connect to the file.
**Rejected: convert Office formats with headless LibreOffice.** It adds
hundreds of MB to both images and a large parser for untrusted input, all
for PPTX, which a bounded `zipfile` text extraction covers.

**Tests (implementation card).**

- A closed-world test walks every entry in `CHAT_FILE_DOCUMENT_EXTENSIONS`
  and `CHAT_FILE_DOCUMENT_MIME_TYPES`. It asserts that each lands in exactly
  one class of the admission table, and that every admitted class has a
  Converse format. A new suffix cannot be admitted without one.
- `tests/unit/test_messaging_modalities.py:365`
  `test_documents_use_real_file_chat_allowlist` keeps `pptx` admitted, and
  asserts `bedrock_format == 'md'` plus `## Slide 1` text from a fixture deck.
- Upload: a 4,500,001-byte document, a 101-page PDF, a non-UTF-8 `.py`, an
  `.rtf`, a PPTX zip bomb and a PPTX with a `DOCTYPE` each get 400 with their
  reason. A 9000 px PNG is stored at 2048 px. A BMP is stored as JPEG with
  `mime_type` `image/jpeg`.
- Answer time: 6 documents produce 5 document blocks and an omission note.
  Block names match `document-\d+`. The 20 MB budget and the 100-page PDF
  budget each stop inclusion. An S3 object larger than its cap raises before
  any provider call.
- Flutter: a 400 with `detail` shows that text in the snackbar. A 500 shows
  the generic text.
- Deployed: one PDF, DOCX, XLSX, PPTX, Markdown, PNG, JPEG, GIF and WebP each
  go through `POST /v2/files`. A `POST /v2/messages` asking about each one
  returns an answer that quotes its content.

#### Private chat thumbnails

Today a thumbnail is a public object. `upload_multi_chat_files`
(`utils/other/storage.py:2343`) uploads `{uid}/{thumbnail_name}`, calls
`blob.make_public()` (2365) and returns `blob.public_url`. The upload routes
store that URL in `u/files/{id}.thumbnail` (`routers/chat.py:1567-1577`,
`utils/media/video_upload.py:39-47`). The chat page renders it with an
unauthenticated `CachedNetworkImage` (`pages/chat/widgets/files_handler_widget.dart:141`).
`BUCKET_CHAT_FILES` blocks public access in this port, so that URL would
return 403. The contract below replaces the public URL with an owner-checked
backend route.

**Object key.** The backend mints the key from the file id, never from the
upload's file name: `{uid}/thumbnails/{file_id}.{ext}`. The row stores it
in a new `thumbnail_key` field on `FileChat` (`models/chat.py:380`). Every key
in the bucket starts with `{uid}/`, so the owner purge that account deletion
already runs on `BUCKET_CHAT_FILES` (`storage.py:260`, prefix `{uid}/`) covers
attachments, filmstrips and thumbnails. Writes carry no ACL and keep the
existing `owner_storage_write_gate`. `upload_multi_chat_files` becomes
`upload_chat_thumbnail(uid, file_id, source_path) -> key`, and all four
callers move together: `routers/chat.py:1568` (`/v2/files`), `:1643` (legacy
`/v1/files`), `utils/media/video_upload.py:39` (video) and
`utils/messaging/adapters/media.py:227` (channels). Minting from the id also
removes a collision: the channel adapter names the thumbnail after the
attachment's own name, so two photos named `image.jpg` overwrite each other's
thumbnail today.

**Thumbnail format.** `File.generate_thumbnail` (`chat_file.py:212-224`)
saves the thumbnail in the source's Pillow format, so the thumbnail format is
the stored image's format from the attach contract above. That is one of
JPEG, PNG, GIF or WebP, because converted images are already PNG or JPEG. A
video poster is always a JPEG (`_write_poster`, 309). One fixed map gives both
the key suffix and the object's `ContentType`:

| Pillow format | `{ext}` | `ContentType` |
| --- | --- | --- |
| `JPEG` | `jpg` | `image/jpeg` |
| `PNG` | `png` | `image/png` |
| `GIF` | `gif` | `image/gif` |
| `WEBP` | `webp` | `image/webp` |

An animated GIF's thumbnail is its first frame, saved as GIF. A format not in
the map raises before any S3 write, which the attach contract makes
unreachable. `upload_chat_thumbnail` sets `ContentType` on `PutObject`.
The route serves the object's own `ContentType`, not the row's.

**Fetch route.** A new `GET /v2/files/{file_id}/thumbnail` in
`routers/chat.py` takes `auth.get_current_user_uid`. It reads the row with
`chat_db.get_chat_files(uid, [file_id])`, so another user's id finds no row.
It returns 404 when there is no row, no `thumbnail_key`, a key outside
`{uid}/`, a key whose suffix is not in the format map, or no object.
Otherwise it streams the object with its stored `ContentType`,
`X-Content-Type-Options: nosniff`, `Cache-Control: private, max-age=86400`
and the object's ETag.
The route needs an entry in `route_policy_manifest.yaml` beside `/v2/files`
(line 87).

**Projection.** The stored `thumbnail` field holds the stable, unsigned URL
`{BASE_API_URL}/v2/files/{file_id}/thumbnail`, which never expires. Every
reader then returns a working value without re-signing anything:

- the upload responses (`routers/chat.py:1586` `client_dump`, `routers/chat_video.py:98`);
- `GET /v2/messages`, which re-attaches rows from `u/files`
  (`database/chat.py:437-449`);
- the human message's embedded `files` copy, which `utils/chat_turn.py:354`
  sets and `_persist_message` (388) writes into `u/messages`;
- `GET /v2/desktop/messages` (`routers/chat_sessions.py:216`), which returns
  that embedded copy as stored;
- the client's own `cachedMessages` list in SharedPreferences
  (`backend/preferences.dart:1365-1375`), which `chat_history_state.dart:280`
  shows when the server list is empty.

The URL must be absolute. `FilesHandlerWidget._isLocalPath` (line 14) treats
a value starting with `/` as a local file path.

**Client.** `FilesHandlerWidget._thumbnailImage` (112) hands network URLs to a
small stateful `ChatThumbnailImage`, modelled on `widgets/omi_map_preview.dart:146-178`.
It resolves `getAuthHeader()` (`backend/http/shared.dart:100`) before the
first fetch and shows the existing fallback tile while the header resolves.
It then passes `httpHeaders: {'Authorization': ...}` and
`cacheKey: 'chat-thumb:<file id>'`, so the image cache survives token
changes. The header is attached only when the URL's origin equals
`Env.apiBaseUrl`'s origin, so the bearer token never goes to another host. On
an error it resolves the header once more (the token may have expired since
the first build) and retries once, then shows the fallback tile.

**Deletion.** Today a failed provider delete is logged and its row deleted
anyway (`utils/other/chat_file.py:628-636`), which orphans the object. Keeping
the failed row alone does not make a retry possible, because cleanup finds
rows only through the session's `file_ids` (624-627), and both cascade routes
delete that session right after cleanup: `DELETE /v2/chat-sessions/{session_id}`
through `delete_chat_session(cascade_messages=True)` (`database/chat.py:892-910`),
and the implicit-session branch of `DELETE /v2/messages` through
`delete_chat_session` (`routers/chat.py:356-357`). The legacy
`DELETE /v1/messages` (`routers/chat.py:1683-1717`, still mounted, not called
by the app) does the same at 1714-1715. Once the session is gone,
nothing points at the surviving rows.

**Decision: the file row is the retry pointer.** Cleanup marks a session's
rows as pending before it touches S3, so the rows stay discoverable after the
session is deleted:

1. **Claim.** `FileChatTool.cleanup` reads the rows for the session's
   `file_ids` (raw dicts, no model validation, as `_openai_file_ids` (94)
   does) and writes `cleanup_pending_at=<now>` on each in one batch. If that
   write fails, cleanup raises and the route answers 500 before any message,
   session or object is deleted, so the client's next delete starts over.
2. **Delete objects.** One S3 `DeleteObjects` call per 1,000 keys, each
   naming the attachment key, `video.strip_file_id` and `thumbnail_key`, all
   under `{uid}/`. A key outside `{uid}/` is skipped and its row kept, so a
   corrupt row cannot delete another user's object.
3. **Delete rows.** Delete only the rows whose every key came back in
   `Deleted`, or as `NoSuchKey`. A row with a key in `Errors`, or a whole call
   that raised, keeps its row and its `cleanup_pending_at`. Cleanup returns
   the count of kept rows and logs it. It does not raise for a partial S3
   failure.
4. **Cascade.** The route then deletes the messages and the session as
   today. The kept rows no longer need the session to be found.

**Retry owner.** `FileChatTool.retry_pending_cleanup(uid, limit=100)` selects
the uid's rows with `cleanup_pending_at` set and runs steps 2 and 3 on them.
It touches only rows that a cleanup claimed, never another session's
attachments. Both cascade routes call it first, best effort (an error is
logged and the route continues), so the next chat clear or session delete
retries earlier failures. Account deletion removes everything left through the
`{uid}/` prefix purge above.

Entry points:

- `DELETE /v2/messages` already calls cleanup (`routers/chat.py:346-353`).
  Its order becomes `retry_pending_cleanup` → `clear_chat` → `cleanup` →
  `delete_chat_session` for the implicit session. The explicit-session branch
  keeps the session and runs the same cleanup.
- `DELETE /v2/chat-sessions/{session_id}` (`routers/chat_sessions.py:172-178`)
  does not call cleanup today, so its rows and objects survive. It becomes
  `retry_pending_cleanup` → `FileChatTool(uid, session_id).cleanup()` →
  `delete_chat_session(cascade_messages=True)`. Cleanup must run before the
  cascade, while the session still lists its `file_ids`.
- `DELETE /v1/messages` (`routers/chat.py:1683`) takes the same order as the
  implicit-session `DELETE /v2/messages`. It is mounted with the retained
  `chat` module, so it must not delete a session without the claim either.
- Account deletion is covered by the `{uid}/` prefix purge above.

**Rejected: keep the session when cleanup partially fails.** The user asked to
delete the chat. A kept session stays in `GET /v2/chat-sessions` and the
history list, and `DELETE /v2/messages` would need a new error the client
does not handle. **Rejected: a separate cleanup queue table.** The rows
already hold the keys and the owner, so a second record would duplicate them
and need its own consistency rule.

**Rejected: a fresh presigned URL on every projection.** The URL is persisted
in three places the backend does not rewrite on read: the embedded message
copy, the desktop history route that returns it, and the client's
`cachedMessages`. Each would hold an expired URL once its signature lapsed.
`CachedNetworkImage` keys on the URL by default, so a new signature per read
would also defeat the image cache. A URL presigned with the task role's
temporary credentials stops working when those credentials expire, even if
the URL's own expiry is later.

**Tests (implementation card).**

- Backend unit: upload writes `thumbnail_key = {uid}/thumbnails/{file_id}.{ext}`
  and the stable URL, with no ACL or `make_public` call, for all four callers.
  Two channel photos with the same name get distinct keys. A test
  parametrized over JPEG, PNG, GIF (animated and still), WebP and a BMP
  source asserts the key suffix and `ContentType` from the format map. The
  BMP lands as `jpg`. A video poster lands as `jpg`.
- Route: the owner gets 200 with bytes and the stored content type for each
  of the four formats, plus `nosniff`. A key with an unmapped suffix gets 404. Another uid's file
  id, a row without a key, a key outside `{uid}/` and a missing object each get
  404. A request without a token gets 401.
- The embedded `files` copy read back from `u/messages` and through
  `/v2/desktop/messages` fetches successfully. Its URL carries no signature or
  expiry parameter.
- Cleanup deletes the attachment, strip and thumbnail objects, then the rows.
  `DELETE /v2/chat-sessions/{id}` runs cleanup.
- Partial failure and retry, parametrized over both cascade entries
  (`DELETE /v2/chat-sessions/{id}`, the implicit-session
  `DELETE /v2/messages` and `DELETE /v1/messages`): session `S` holds files `f1` and `f2`, and the
  same uid has file `g` in session `T`. A stubbed S3 client returns `f1`'s
  keys in `Deleted` and `f2`'s thumbnail key in `Errors`. After the route,
  `S` and its messages are gone, `f1`'s row is gone, and `f2`'s row exists
  with `cleanup_pending_at` set. `g`'s row and objects are untouched and
  unmarked. The next cascade call on `T` first retries: the stub now deletes
  `f2`'s keys, `f2`'s row is gone, and `g` is handled by `T`'s own cleanup.
- A failed claim write answers 500, and the session, messages, rows and
  objects all remain. A row whose key is outside `{uid}/` keeps its row and
  sends no `DeleteObjects` entry for that key.
- Against the deployed bucket, an anonymous GET of the object's S3 URL returns
  403 and the stable route returns 401.
- Flutter widget test (`test/pages/chat/`): an API-origin URL gets the
  `Authorization` header and the `chat-thumb:<id>` cache key. A foreign-origin
  URL gets no header. A 401 re-resolves the header once and then falls back.

### Valkey: TLS on every client

`omi-valkey` has in-transit encryption `required` and an AUTH token.
ElastiCache allows AUTH only with in-transit encryption on, so a plaintext
client gets no connection. Production code builds Redis clients in several
places, and most of them never call `redis_connection_kwargs`. Adding `ssl`
to that helper alone would leave the bypass constructors failing on every
call. Each one catches the error and degrades quietly, so the loss would show
up only as wrong routing, admission or cache behaviour.

**Constructor census.** Every `Redis(` constructor outside `tests/` and
`testing/` (grep for `Redis(`, `StrictRedis(`, `from_url(`,
`ConnectionPool(`, `RedisCluster(`, `Sentinel(`, plus the aliased imports
`aioredis` and `from redis.asyncio import Redis`):

| Constructor | Kind | Reached in the MVP | Without TLS against `omi-valkey` |
| --- | --- | --- | --- |
| `database/redis_db.py:26` `r` | sync, through `redis_connection_kwargs` | yes: rate limits, auth fences, signed-URL cache, `/ready` | every call fails. `/ready` reports 503 |
| `database/redis_db.py:713` `get_async_redis_client` | async, through `redis_connection_kwargs` | yes: listen registry (`routers/listen/registry.py:56`) | listen bookkeeping fails |
| `database/redis_connection.py:20` `create_bounded_redis_client` | sync, through `redis_connection_kwargs` | yes: 22 call sites, among them `omi_flags`, `action_item_versions`, `rollout_membership_cache`, `fleet_telemetry` and `final_pass_sync` | each caller's fallback |
| `utils/stt/live_health.py:237` `_redis` | async, direct | yes: `main.py:242` starts `refresh_forever` on every boot; `live_chain.py`, `live_session.py` and `streaming.py` read it | the fleet snapshot never refreshes. `has_fresh_fleet_snapshot` stays false and each task routes on its own local scores |
| `utils/translation_core/cache.py:244` `_create_redis_client` | sync, direct | yes: every `/v4/listen` session builds a `TranslationService` (`routers/listen/transcripts.py:184`), which uses `get_default_translation_store` (`utils/translation.py:72`) when a translation language is set | `RedisError` is caught (78, 115, 125, 138), so every translation misses the cache and calls the LLM again |
| `utils/stt/demand_presence.py:91` | async, direct | conditional: demand routing (`routers/listen/demand_routing.py:134`) when the managed chain and session recovery are on | presence reports `unavailable` and keeps visible admission |
| `utils/stt/paid_admission.py:71` | async, direct | conditional: `STT_PAID_SPILLOVER_BUDGET_ENABLED` (default `false`, line 39) | every promotion is denied as `unavailable` |
| `utils/llm/vertex_reservation_state.py:75` | async, direct | no: Vertex reservations (desktop proxy, dream capacity, LLM gateway) | n/a in the MVP |
| `database/proactivity_redis.py:14` `_client` | sync, direct, own endpoint | no: only when `PROACTIVITY_REDIS_HOST` is set. Otherwise it returns `redis_db.r` | n/a in the MVP |
| `scripts/stt/reset_fleet_provider.py:39`, `scripts/reset_live_stt_cost_gate.py:23` | operator scripts, direct | run by hand against the serving environment | the script cannot connect |
| `scripts/redis/census.py:56` | operator script, `redis-cli` subprocess | run by hand | needs `--tls` |

**Seam.** `database/redis_connection.py` gains one function and one setting:

- `REDIS_DB_TLS=true` in the task environment. Unset or `false` keeps
  today's plaintext behaviour, so upstream and local stacks do not change.
- `redis_tls_kwargs() -> dict` returns `{}` when the switch is off. When it
  is on, it returns `ssl=True`, `ssl_cert_reqs='required'`,
  `ssl_check_hostname=True` and `ssl_ca_certs=certifi.where()`. The keys are
  the same for `redis.Redis` and `redis.asyncio.Redis` in redis 5.0.8.
- `redis_connection_kwargs` merges `redis_tls_kwargs()`, so `r`,
  `get_async_redis_client` and `create_bounded_redis_client` pick it up.

The explicit `ssl_check_hostname=True` is required. redis 5.0.8 defaults it
to `False` in both the sync `SSLConnection` and the async `RedisSSLContext`.
That default would encrypt the connection without checking that the
certificate names the endpoint. The endpoint's certificate is issued for
`*.cache.amazonaws.com`, so `REDIS_DB_HOST` must be the primary endpoint DNS
name, not an IP. `certifi==2026.10.18` is already pinned in
`backend/requirements.txt:23` but not in `pusher/requirements.txt`. The pusher
image copies `database/` (`pusher/Dockerfile:47`) and pins `redis==5.0.8`, so
`certifi` goes into both files (see "Python dependency closure").

Each direct constructor in the census keeps its own timeouts, retry and
`decode_responses` settings and adds the shared connection fields:

- `live_health.py:237`, `demand_presence.py:91`, `paid_admission.py:71` and
  `translation_core/cache.py:244` build from `redis_connection_kwargs()` and
  then override their timeouts. Their host, port and password reads move into
  the helper. These three async modules default the host to `localhost` when
  `REDIS_DB_HOST` is unset, so the helper keeps that fallback for them through
  a `default_host` argument.
- `vertex_reservation_state.py:75` adds `**redis_tls_kwargs()`. It stays off
  the MVP path, but it reads the same `REDIS_DB_*` endpoint, so it must not
  break when a later card enables it.
- `proactivity_redis.py:14` is the one intentionally distinct endpoint. It
  keeps `PROACTIVITY_REDIS_HOST`/`_PORT`/`_PASSWORD` and gains its own
  `PROACTIVITY_REDIS_TLS` switch, read through the same helper with a prefix
  argument. It does not inherit `REDIS_DB_TLS`, because its endpoint may be a
  different service.
- The two Python scripts add `**redis_tls_kwargs()`. `scripts/redis/census.py`
  passes `--tls` to `redis-cli` when `REDIS_DB_TLS` is true.

**Guard.** A unit test scans `backend/` outside `tests/` and `testing/` for the
constructor patterns above and fails on any match outside an allowlist of the
named call sites, each of which must reference `redis_tls_kwargs` or
`redis_connection_kwargs`. A new direct constructor therefore fails CI
instead of silently connecting in plaintext.

**Tests (implementation card).**

- Unit: with `REDIS_DB_TLS=true`, each named constructor (sync `r`, async
  `get_async_redis_client`, `create_bounded_redis_client`, `live_health`,
  `demand_presence`, `paid_admission`, `translation_core.cache`,
  `vertex_reservation_state`) builds a client whose connection pool uses
  `SSLConnection` (sync) or carries an `ssl_context` with
  `check_hostname=True` and `verify_mode=CERT_REQUIRED` (async). With the
  switch unset, each builds a plaintext pool, as today.
- Unit: `PROACTIVITY_REDIS_HOST` set with `REDIS_DB_TLS=true` and
  `PROACTIVITY_REDIS_TLS` unset builds a plaintext client to the proactivity
  host. The two endpoints stay separate.
- Integration: a local TLS Valkey container (`--tls-port`, a test CA,
  `--requirepass`) accepts each constructor with the test CA, and refuses a
  client built with the switch off. A certificate whose name does not match
  the host is refused.
- Deploy check: after the first boot, `/ready` returns 200,
  `live_stt_health.has_fresh_fleet_snapshot()` becomes true within one
  refresh period, and a repeated translation of the same segment reads the
  cache.

**Postgres, same rule.** The two Postgres constructors take a full DSN when
one is set: `database/jobstore/connection.py:46` `dial` (`psycopg.connect(dsn)`)
and `database/cells/serving.py:22` (`CELLS_PG_DSN`). Only the GCP
`iam_connector` (65) sets `sslmode`. The DSN built from the RDS secret for the
adapter, `CELLS_PG_DSN` and `JOBSTORE_PG_*` therefore carries
`sslmode=verify-full` with `sslrootcert` set to the RDS CA bundle baked into
the image, and the cluster parameter group sets `rds.force_ssl=1`. A
plaintext DSN is then refused by the server rather than accepted.

### Hosting

| Resource | Name / value |
| --- | --- |
| VPC | `omi-vpc`, `10.42.0.0/16`, 2 AZs. Avoids the existing `10.0.0.0/16` (x2) and `172.31.0.0/16` |
| Subnets | `omi-public-a/b` (tasks, ALB), `omi-isolated-a/b` (Aurora, Valkey). No NAT gateway |
| VPC endpoint | S3 gateway endpoint on the public route tables |
| ECS cluster | `omi-cluster` (Fargate, x86_64). Existing clusters `pihole-cluster` and `KeycloakStack` are untouched |
| ECS service / task | `omi-backend`. One task with containers `backend` (port 8080) and `pusher` (port 8081). Starting size 2 vCPU and 4-6 GiB (`UNVERIFIED`; measure the image's resident size first). Both images bind 8080 by default, so the `pusher` container definition overrides `command`, as described under "Pusher sidecar" |
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

**Pusher sidecar.** Platform-key conversation finalization does not run on
pusher in this port. It runs on the backend container's in-process worker (see
"Durable finalization"), because with durable dispatch enabled
`request_finalization` routes platform-key jobs to the worker and skips the
pusher handoff (`utils/conversations/lifecycle.py:1246-1262`). Pusher still
runs, as a sidecar in the same task, for the work `/v4/listen` streams to it
over `/v1/trigger/listen` (`routers/pusher.py:2040`): the private-cloud-sync
audio archive (`upload_audio_chunks_batch`), transcript webhooks, speaker
samples, and BYOK finalization. The backend reaches it at
`HOSTED_PUSHER_API_URL=http://localhost:8081`. Pusher stays off the ALB,
because `/v1/trigger/listen` takes a `uid` query parameter without
authenticating it. A pusher restart can drop live archive frames for the
sessions open at that moment. It cannot strand a conversation, because no
finalization job depends on pusher.

Fargate tasks use `awsvpc` networking, so both containers share one network
namespace and one port space. Both images hard-code `--port 8080` in `CMD`
(`backend/Dockerfile:63`, `backend/pusher/Dockerfile:58`), so pusher cannot
keep its image default. The `pusher` container definition overrides `command`
with the image's arguments, changing only the port:

```json
["uvicorn", "pusher.main:app", "--host", "0.0.0.0", "--port", "8081",
 "--ws-ping-interval", "20", "--ws-ping-timeout", "20", "--loop", "uvloop",
 "--timeout-graceful-shutdown", "90"]
```

Its `portMappings` entry is `containerPort: 8081`, and its container health
check is `curl -fsS http://localhost:8081/health`. The image is unchanged, so
upstream's deployment keeps port 8080. The hosting card adds a synth-time
assertion that the `pusher` command contains `--port 8081` and that
`HOSTED_PUSHER_API_URL` names the same port.

**S3 buckets** (names are `omi-<purpose>-717494614307-apse2`; versioning off,
SSE-S3, public access blocked):

| Env var | Bucket | MVP use |
| --- | --- | --- |
| `BUCKET_PRIVATE_CLOUD_SYNC` | `omi-private-cloud-sync-717494614307-apse2` | live audio archive (private cloud sync defaults on) |
| `BUCKET_CANONICAL_AUDIO` | `omi-canonical-audio-717494614307-apse2` | canonical audio for finalization |
| `BUCKET_AUDIO_DERIVED` | `omi-audio-derived-717494614307-apse2` | derived audio artifacts |
| `BUCKET_SPEECH_PROFILES` | `omi-speech-profiles-717494614307-apse2` | `GET /v3/speech-profile` existence check |
| `BUCKET_CHAT_FILES` | `omi-chat-files-717494614307-apse2` | chat attachments, filmstrips and thumbnails, all under `{uid}/` and served only through owner-checked routes (see "Private chat thumbnails") |
| `BUCKET_POSTPROCESSING` | `omi-postprocessing-717494614307-apse2` | reprocess inputs |

Other `BUCKET_*` variables (`BUCKET_BATCH_PIPELINE`, `BUCKET_SCREEN_FRAMES`,
`BUCKET_FRAME_REQUESTS`, `BUCKET_FRAME_REQUESTS_TEMPORARY`,
`BUCKET_TEMPORAL_SYNC_LOCAL`, `BUCKET_MEMORIES_RECORDINGS`,
`BUCKET_DESKTOP_UPDATES`, `BUCKET_PLUGINS_LOGOS`, `BUCKET_APP_THUMBNAILS`) stay
unset for MVP. Leaving a bucket unset does not remove its routes. Every route
that reads one of these buckets is also removed by a cut switch: `screen_frames`
for screenshots, `frame_requests` for photo images, and `apps` for thumbnails
and logos.

GCS semantics in the S3 client: `if_generation_match=0` becomes
`If-None-Match: *`. Other generation preconditions become `If-Match` on the
ETag plus a metadata counter, which is a correctness risk to test. Resumable
uploads become multipart uploads.

**No public object URLs.** Every bucket blocks public access, so no storage
helper may return an object's bare URL to a client. `blob.public_url`,
`_blob_public_url` (`storage.py:149`) and `make_public` have no S3 meaning in
the new client: `public_url` raises and `make_public` is not implemented, so
a caller that still relies on them fails in tests instead of returning a
403 URL. On the MVP path the only such caller is the chat thumbnail, replaced
above. `upload_profile_audio` (309) also returns a public URL, but its route
`POST /v3/upload-audio` is cut. The others write buckets that stay unset.

**Presigned URLs.** GCS signed URLs become S3 presigned GET URLs. Two
details differ:

- A presigned URL signed with temporary credentials stops working when those
  credentials expire, whatever its own expiry says. The ECS task role's
  credentials rotate, so `_get_signed_url` (`storage.py:2284`) presigns with
  the task role and caps each URL's lifetime at the minimum of the requested
  minutes and the remaining credential lifetime minus 5 minutes.
- `_get_signed_url` caches each URL in Redis for its full lifetime (2296,
  `redis_db.cache_signed_url`). The cache TTL becomes that capped lifetime
  minus 60 s, so a cached URL is never handed out in its last minute. The
  same applies to the screen-frame path (2420-2427), which is cut.

The MVP routes that return presigned URLs are `GET /v1/users/people`,
`GET /v1/users/people/{person_id}` (speech samples, 60 minutes) and
`GET /v1/sync/audio/{conversation_id}/urls` (playback, 60 minutes). The
playback URLs are fetched on each play and not stored. The people list is
stored in `cachedPeople` (`backend/preferences.dart:1429-1437`), but its
sample URLs are played only from `pages/settings/person_detail_page.dart:137`,
which is outside the MVP, and `PeopleProvider.setPeople` (74-92) replaces the
cache with a fresh fetch on every load. The people settings card that brings
that page back must refresh before playing, or re-fetch on a 403. A unit test
presigns with credentials that expire in 10 minutes and asserts the URL and
its cache TTL both end before then.

**Secrets Manager entries** (prefix `omi/`):

| Secret | Consumer |
| --- | --- |
| `omi/encryption-secret` | `ENCRYPTION_SECRET` (per-uid HKDF key) |
| `omi/memory-v3-cursor-secret` | `MEMORY_V3_CURSOR_SECRET`, required by `utils/memory/universal_list_cursor.py` |
| `omi/sync-content-id-secret` | `SYNC_CONTENT_ID_SECRET` |
| `omi/valkey-auth` | `REDIS_DB_PASSWORD` (the AUTH token, sent as user `default`) |
| RDS-managed master secret for `omi-aurora` | Postgres DSN for the adapter, `CELLS_PG_DSN`, `JOBSTORE_PG_*`, each with `sslmode=verify-full` |
| `omi/github-token` (optional) | `GITHUB_TOKEN` for firmware release reads |

### Durable finalization

Upstream has two listen-finalization modes (`backend/docs/listen_finalization_jobs.mdx`,
"Configuration and rollout"). `inline` hands each job to pusher over the live
WebSocket. `cloud_tasks` wakes a dedicated worker through a Cloud Tasks queue.
In both modes the job row in `conversation_finalization_jobs`, with its
`dispatch_generation` and lease epoch, is the source of truth, and the queue
only carries the opaque wake `{job_id, dispatch_generation}`. Every durable
recovery path is gated on the `cloud_tasks` mode or on a configured Cloud Tasks
binding, so `inline` silently disables all of them:

| Recovery path | Gate | Effect in `inline` |
| --- | --- | --- |
| Stale `in_progress` admission, `reconcile_stale_in_progress_conversations` | `services/conversation_finalization.py:124`, mode check | Returns `mode: off`. A content-bearing conversation whose live session died is never finalized |
| Queued and leased job replay, `reconcile_listen_finalization_jobs` | `services/conversation_finalization.py:231-232`, `replay_enabled()` and `is_listen_finalization_tasks_configured()` | Returns before scanning. A queued job whose pusher handoff was lost stays queued |
| Self-heal admission | `services/conversation_selfheal.py:598-607`, `request_finalization(..., require_cloud_tasks=True)` | Raises `FinalizationDispatchUnavailable`, logged as `dispatch_unavailable` |
| Quiet finalization checks | `utils/conversations/quiet_finalization.py:34` (wake) and `:100` (run) | No wake is sent, and a replayed check raises |
| Merge recovery, `merge_conversations_job.reconcile` | `utils/conversations/merge_conversations_job.py:76` calls `enqueue_listen_finalization_job` directly | Raises, because the binding is empty |
| `POST /v1/conversations` | `routers/conversations.py:512` | Takes the legacy synchronous path instead of the exact-ID outbox (`finalize_conversation`, `routers/conversations.py:629`) |

The port therefore keeps the durable design and replaces only the transport.

**Decision.** Add a third mode, `LISTEN_FINALIZATION_DISPATCH_MODE=in_process`.
The backend container runs one bounded asyncio dispatcher. It replaces the
Cloud Tasks hop and keeps every other part of the durable path.

| Seam | Change |
| --- | --- |
| `utils/cloud_tasks.py:328` `is_listen_finalization_dispatch_enabled` | True for `cloud_tasks` or `in_process` |
| `:341` `is_listen_finalization_dispatch_configured`, `:346` `is_listen_finalization_tasks_configured` | In `in_process`, true with no GCP binding. The `cloud_tasks` regex checks are unchanged |
| `:649` `enqueue_listen_finalization_job` | In `in_process`, hands `(job_id, dispatch_generation, schedule_at)` to the dispatcher instead of `enqueue_named_task`. This is the only enqueue seam: live admission (`lifecycle.py:1262`), the reconcilers (`services/conversation_finalization.py:414`), quiet checks (`quiet_finalization.py:37`) and merge recovery (`merge_conversations_job.py:76`) all call it, so none of them changes |
| `routers/conversation_finalization.py:106` `run_listen_finalization_job` | Extract the body after payload parsing into `handle_listen_finalization_wake(job_id, dispatch_generation, retry_count)`, returning the same status and body. The HTTP route keeps calling it. Its OIDC dependency answers 403 when `LISTEN_FINALIZATION_TASKS_HANDLER_URL` is unset (`verify_configured_cloud_tasks_oidc`, `utils/cloud_tasks.py:669`), so the route stays closed on the ALB |
| New `utils/conversations/finalization_dispatcher.py` | A delay heap and an `asyncio.Semaphore(2)`, started from the `main.py` lifespan beside `_periodic_listen_finalization_reconcile` (278). A wake with `schedule_at` waits until that instant. A 2xx result drops the wake. A 409 or 500 re-arms the same wake after 30 s, 60 s, 120 s, then 300 s, until `LISTEN_FINALIZATION_TASKS_MAX_ATTEMPTS` (default 5) is reached. After that the wake is dropped, and the reconcilers own the job. The retry count passed to the handler is the in-memory attempt number. The handler already takes `max(task_retry_count, failed_attempts)` from the job row (`routers/conversation_finalization.py:82`), so a restart that resets the counter does not reset the budget |
| Shutdown | The dispatcher stops taking wakes when lifespan shutdown starts. Wakes still in its heap are lost on purpose, because the reconcilers replay them |
| `scripts/runtime_env_durable_dispatch_contracts.py` | Unchanged. It enforces `cloud_tasks` for `prod` only, and the fork runs `OMI_ENV_STAGE=dev` |

Duplicate wakes are safe for the same reasons they are under Cloud Tasks. The
handler checks `dispatch_generation`, takes the per-job Redis run lock
(`try_acquire_job_run_lock`), and claims a lease epoch in a transaction, so a
wake from the dispatcher and a replay from the reconciler cannot both run one
job.

**Loss bounds.** Producers, the dispatcher and the reconcilers are all in the
backend container. No wake producer runs in pusher.

| Failure | Recovery | Bound |
| --- | --- | --- |
| Wake lost before it runs (container restart, dispatcher crash, attempts exhausted) | The startup drain (`main.py:386`, `_drain_listen_finalization_jobs`) or the periodic reconciler re-enqueues a queued job once it is older than `LISTEN_FINALIZATION_RECONCILE_STALE_SECONDS` (default 300, `database/conversation_finalization_jobs.py:85`) | About 6 minutes: 300 s staleness plus one 60 s tick |
| Container killed while a job holds its lease | The lease expires after `DEFAULT_LEASE_SECONDS=1500` (`database/conversation_finalization_jobs.py:84`), then the reconciler reclaims it with a new epoch | About 26 minutes. This is the same bound Cloud Tasks gives, because its dispatch deadline is also 1500 s |
| Live session lost before admission (backend restart, socket drop) | `reconcile_stale_in_progress_conversations` admits the content-bearing `in_progress` row with `SERVER_RECOVERY` through the 5-minute content-due scan, and its legacy cursor handles markerless rows after `STALE_AFTER` (2 h) | 5 to 6 minutes for indexed rows. 2 hours for markerless rows |
| Quiet check wake lost | `_reconcile_quiet_candidate` (`services/conversation_finalization.py:173`) replays due checks from their indexed due instant | One reconcile tick after the due instant |
| Repeated handler failure | The existing budget: one delayed requeue, then the job row records `dead_letter` | Five attempts, as upstream |
| Aurora data loss | None. The job rows are the record | Out of scope. Aurora backups apply |

No admitted job is lost while Aurora keeps its rows. The cost is latency after
a restart, and the bounds above are the accepted latency.

**Rejected: SQS.** A standard queue plus a DLQ and a long-poll consumer in the
backend container would replace Cloud Tasks one to one. It needs a
15-minute `DelaySeconds` cap with self-deferral for later quiet checks, an
1800 s visibility timeout above the 1500 s deadline, and IAM for five SQS
actions. With one task, the consumer runs in the same container as the
producers and the reconcilers. When that container dies, the queued messages
wait for the same restarted container that the reconcilers would replay from.
SQS therefore adds a service, a DLQ to inspect and an IAM surface without
shortening any bound above. Revisit when the backend runs more than one task,
or finalization moves to its own service. Then the extracted handler is the
SQS consumer's body, and only `enqueue_listen_finalization_job` changes again.

**Other paths that call Cloud Tasks directly.** These were checked for the
same gap. None is on the MVP path in `in_process` mode:

- Demand processing: `postpone` and `_wake` call `enqueue_named_task`
  (`utils/conversations/demand_processing.py:92`, `:276`). It stays off
  (`DEMAND_PROCESSING_ENABLED` defaults false, `config/demand_processing.py:12`),
  so `schedule` returns `False` before any wake (`demand_processing.py:116`).
- Enrichment outbox (`utils/enrichment/outbox.py:288`, `:334`): it owns no
  runtime outside `prod` or the Firestore emulator, so it is inert in `dev`.
- Final-pass retry (`utils/conversations/final_pass_retry.py:44`),
  provider-STT wakes (`utils/provider_stt_tasks.py:22`) and quarantine recovery
  (`utils/batch_pipeline/quarantine_recovery.py:715`): batch-pipeline paths
  reached only from the cut sync uploads.
- Account deletion and conversation batch purge: both run in process when
  their Cloud Tasks binding is unset. Account deletion requires Cloud Tasks
  only for `prod` (`validate_account_deletion_dispatch_configuration`,
  `utils/cloud_tasks.py:200`), and the batch purge stays inline without a
  binding (`utils/conversations/deletion_batch_purge.py:54`).
- Cloud Scheduler jobs (`backend/deploy/scheduler/jobs.yaml`): none is on the
  MVP path.

**Tests for the implementation card:**

1. Unit tests beside `tests/unit/test_listen_finalization_cloud_tasks.py`: with
   `in_process` and no GCP binding, the three predicates are true,
   `request_finalization(require_cloud_tasks=True)` admits, and
   `enqueue_listen_finalization_job` reaches the dispatcher, not
   `enqueue_named_task`.
2. Dispatcher tests: `schedule_at` is honoured; 2xx drops the wake; 409 and 500
   re-arm with the stated backoff; wakes stop at the attempt cap; a duplicate
   wake for a leased job gets 409 and does not run the job twice.
3. Reconciler tests extending `test_conversation_finalization_jobs*.py`,
   `test_conversation_quiet_finalization.py` and `test_selfheal_*.py`:
   `reconcile_stale_in_progress_conversations` no longer returns `mode: off`,
   and `reconcile_listen_finalization_jobs` scans and re-enqueues a stale
   queued job in `in_process`.
4. Restart tests in `testing/listen_pusher_stack/`. `Stack` accepts only
   `inline` and `cloud_tasks` today (`run.py:165`), so add `in_process`. Then:
   (a) admit a conversation, call `restart_backend()` (`run.py:444`) before the
   wake runs, and assert the startup drain completes the job; (b) call
   `restart_pusher()` (`run.py:425`) during a session and assert finalization
   still completes; (c) end a live session without admission, restart the
   backend, and assert the stale `in_progress` scan finalizes it. The fixture
   shortens `LISTEN_FINALIZATION_RECONCILE_STALE_SECONDS` to its 30 s floor and
   the reconcile interval.

### Runtime settings

- `OMI_ENV_STAGE=dev`. `utils/env_loader.py:25` accepts only `prod`, `dev`,
  `local` and `offline`, and `utils/sync/stage.py::current_stage` raises when it
  is unset.
  - `prod` makes `validate_account_deletion_dispatch_configuration`
    (`utils/cloud_tasks.py`, called at `main.py:249`) require Cloud Tasks.
  - `local` and `offline` turn GCS writes into no-ops in `utils/other/storage.py`.
  - `dev` costs four effects: it skips Stripe price validation, prefixes sync
    Redis keys and collections with `dev`, turns on stage profiling and parity
    telemetry, and lets `OMI_LLM_GATEWAY_FEATURE_MODE` take effect without the
    prod override. That variable stays unset, so the last effect does not apply.
  - This choice is `UNVERIFIED` until the first deploy boots clean.
- `MEMORY_ENABLED=on`. Memory fails closed when this is unset
  (`config/memory_rollout.py`).
- `MARKETPLACE_APP_REVIEWERS=<owner sub>` for the transcription and processing
  allowance. The chat quota does not honour reviewers, so seed the owner's
  `users/{sub}` document with `subscription.plan=unlimited`,
  `subscription.status=active` and a far-future `current_period_end`.
- `OMI_ROLE` unset, so `app_roles.py` mounts every router role, then prunes the
  cut routers and routes named by `OMI_DISABLED_ROUTERS` and
  `OMI_DISABLED_ROUTES` (see "Cut switches").
- `DIRECTORY_ROUTING_MODE` unset (off), `FAIR_USE_ENABLED=false`,
  `STT_CAP_DISPATCH_ENFORCEMENT=false`.
- `EMBEDDINGS_PROVIDER=bedrock`, so `utils/llm/clients.py` builds
  `_TitanEmbeddings` as the default embeddings client.
- `MODEL_QOS=aws`, so every feature resolves to `bedrock` and `get_llm` takes
  the explicit Bedrock dispatch (see "Managed dispatch: Bedrock before the
  gateway branch"). The startup check refuses to boot if the name does not
  resolve to the `aws` profile.
- `OMI_LLM_GATEWAY_URL`, `OMI_LLM_GATEWAY_SERVICE_TOKEN`,
  `OMI_LLM_GATEWAY_FEATURE_MODE`, `OMI_LLM_CHAT_AGENT_ROUTE` and
  `OMI_BACKGROUND_FLEX_CAPABLE` stay unset. No gateway runs, and the "Direct
  gateway callsites" table shows that no MVP path reaches one.
- `OMI_SHAPED_AGENT_MODE=on`, as upstream prod sets it
  (`charts/backend-listen/prod_omi_backend_listen_values.yaml:399-400`,
  `charts/pusher/prod_omi_pusher_values.yaml:373-374`). With the default
  `off`, `get_conversation_notes` raises before any model call
  (`utils/llm/conversation_processing.py:461-462`), so conversation structure
  fails for every conversation. `on` also makes `POST /v2/messages`, which
  passes `shaped_invocation=True` (`utils/chat_turn.py:664`), run
  `utils/retrieval/shaped_chat.py::run_shaped_chat_stream` through
  `_run_routed_chat_stream` (`agentic.py:1779-1789`, 1394-1395) instead of
  `_run_openai_agent_stream`. Both runners bind through
  `request_options.bind_chat_tools`.
- `COGNITO_USER_POOL_ID=ap-southeast-4_jA7oAYYrp` and
  `COGNITO_REGION=ap-southeast-4` for the user lookup adapter.
- `REDIS_DB_TLS=true`, `REDIS_DB_HOST` set to the `omi-valkey` primary
  endpoint DNS name, `REDIS_DB_PORT=6379`. `PROACTIVITY_REDIS_HOST` stays
  unset, so proactivity shares that client (see "Valkey: TLS on every
  client").
- `LISTEN_FINALIZATION_DISPATCH_MODE=in_process` (see "Durable finalization").
  `LISTEN_FINALIZATION_REPLAY_ENABLED` stays unset (true,
  `config/listen_finalization.py:6`). `LISTEN_QUIET_FINALIZATION_ENABLED`
  stays unset (true). `LISTEN_FINALIZATION_TASKS_HANDLER_URL`,
  `LISTEN_FINALIZATION_TASKS_QUEUE`, `SYNC_TASKS_PROJECT` and
  `SYNC_TASKS_LOCATION` stay unset, which keeps the Cloud Tasks worker route
  answering 403.
- `DEMAND_PROCESSING_ENABLED` unset (false). Demand processing wakes Cloud
  Tasks directly.
- `LOCAL_PREVIEW_BATCH_ENABLED=false` and `MOBILE_STANDARD_BATCH_ENABLED`
  unset (false, `utils/sync/mobile_standard_batch.py:10`). Both lanes defer the
  live stream and journal audio for the batch upload routes, which are cut.
  `LOCAL_PREVIEW_BATCH_ENABLED` defaults on (`config/cloud_live_stt.py:56`), so
  it must be set.
- `EPISODES_CLIENT_CHANNELS=beta,store`. The default `beta` admits only
  non-production builds (`config/episodes.py:18-25`). Setting both channels
  gives the fork's debug and release builds the same Home episode UI.
  `EPISODES_ENABLED` stays unset (on).
- `REVIEW_SURFACE_MODE` and `DREAM_AGENT_ENABLED` stay at their defaults (off).
  Their routers are also removed by `OMI_DISABLED_ROUTERS`, so the 404 the
  client reads as "Review off" does not depend on the flag.

### Cut switches

Every cut feature is disabled by a named switch. A feature is either
**disabled**, meaning its routes are not mounted and return 404, or
**unconfigured**, meaning it stays mounted but has no credentials, so its
callers get an empty result or a 5xx. The MVP uses disabled for every cut route
the app can reach, and unconfigured only for internal calls with no route.
A router switch removes a whole module. A cut route that lives in a retained
module (for example the calendar-event routes in `conversations`) gets its own
route switch entry.

**Two new switches**, owned by `backend/app_roles.py` and read where
`main.py:148-149` resolves the role and mounts routers:

| Switch | Value | Enforcement |
| --- | --- | --- |
| `OMI_DISABLED_ROUTERS` | Comma-separated `Mount.module` names from `ROUTER_MOUNTS`, for example `apps,payment` | `mounts_for` (`app_roles.py:321`) drops matching mounts before `mount_routers` (329) includes them |
| `OMI_DISABLED_ROUTES` | Semicolon-separated `METHOD /path` entries, matching the decorator path exactly, for example `POST /v1/conversations/search` | After `mount_routers`, remove each matching `APIRoute` (or `APIWebSocketRoute` for `WS /path`) from `app.router.routes` |

Both default to empty, so an unset deployment mounts exactly what it mounts
today and `tests/unit/test_app_roles_route_census.py` stays green. An entry that
matches no mount or route raises `RoleConfigError` at startup, the same way
`role_from_env` (308) rejects an unknown `OMI_ROLE`, so a typo cannot leave a
feature silently mounted. A disabled route returns 404, or 405 when another
method is still mounted on the same path.

**Cut list:**

| Feature | Switch | Disabled behaviour | Client result |
| --- | --- | --- | --- |
| Firebase sign-in (`/v1/auth/*`) | `OMI_DISABLED_ROUTERS=auth` | 404 on `/v1/auth/authorize`, `/callback/*`, `/token`, `/local-dev/custom-token` | Never called. The app uses the Cognito hosted UI. Local and test sign-in use the paths in "Client: non-production auth paths" |
| Firebase-JS consent pages: app-integration OAuth (`routers/oauth.py`) and MCP OAuth authorize (`routers/mcp_sse.py:193`, `:219`) | `OMI_DISABLED_ROUTERS=oauth`, plus `OMI_DISABLED_ROUTES=GET /authorize;POST /authorize` | 404 on `GET /v1/oauth/authorize`, `POST /v1/oauth/token`, `GET /authorize` and `POST /authorize` | Never called by the app. Third-party app OAuth needs the cut app store. MCP clients keep MCP API keys (`/v1/mcp` stays mounted); MCP OAuth sign-in returns with a Cognito consent page later |
| Stripe billing | `OMI_DISABLED_ROUTERS=payment,retention_offer,yearly_switch,redeem_code,storefront`, plus `STRIPE_API_KEY` and `STRIPE_*_PRICE_ID` unset | 404 on `/v1/payments/*`, `/v1/stripe/*`, `/v1/payment-methods/*` | `GET /v1/users/me/subscription` stays mounted and returns the seeded `unlimited` plan without calling Stripe. `showSubscriptionUI` is false, so the paywall never opens |
| Plugins / app store | `OMI_DISABLED_ROUTERS=apps,developer,developer_key` | 404 on `/v1/apps*`, `/v2/apps*` | Conversation detail keeps its cached summary templates (`SummaryTemplatesLoad.failed`) |
| Typesense search | `OMI_DISABLED_ROUTES=POST /v1/conversations/search`, plus `TYPESENSE_HOST` and `TYPESENSE_API_KEY` unset | 404 | Search returns `ConversationSearchResult.failure` (`app/lib/backend/http/api/conversations.dart:1312`). With the env vars unset, `utils/conversations/typesense_index.py` skips index writes |
| Pinecone | `PINECONE_API_KEY`, `PINECONE_INDEX_NAME` unset (unconfigured, no route) | Index is `None`. Queries return empty. Callers that embed before writing or querying check `vector_index_configured()` first, so no Titan call is made (see "Embeddings: Titan input bound and the cut index") | Chat retrieval skips the vector branch (`utils/retrieval/tool_services/conversations.py:330`) |
| Twilio phone calls | `OMI_DISABLED_ROUTERS=phone_calls`, plus `TWILIO_*` unset | 404 on `/v1/phone/*` | `/v4/listen?source=phone_call` cannot start, because the client needs `POST /v1/phone/token` first |
| Telegram / iMessage messaging | `OMI_DISABLED_ROUTERS=messaging`, plus `OMI_TELEGRAM_ENABLED=off` and `OMI_IMESSAGE_ENABLED=off` (the defaults, `utils/messaging/adapters/runtime.py:23,35`) | 404 on `/v1/messaging/*`. `start_adapters` returns `None` | Channel chats page unreachable |
| Google Calendar and X | `OMI_DISABLED_ROUTERS=google_calendar,calendar_onboarding,calendar_meetings,x_connector`, plus `OMI_DISABLED_ROUTES=POST /v1/conversations/{conversation_id}/calendar-event;DELETE /v1/conversations/{conversation_id}/calendar-event;POST /v1/conversations/{conversation_id}/calendar-event/auto-link` | 404 on `/v1/calendar/*` (including `/v1/calendar/meetings*`, a separate module), `/v1/x/*`, and the three conversation calendar-event routes, which live in the retained `conversations` module | Auto-link returns null on any failure. "Link event" asks to connect Google Calendar, which never connects |
| Task and app integrations (Google Tasks, Todoist, Asana, ClickUp, Gmail) | `OMI_DISABLED_ROUTES=GET /v1/task-integrations/{app_key}/oauth-url;GET /v1/integrations/{app_key}/oauth-url`, plus provider client ids unset | 404 on the OAuth starts. The status reads stay mounted and return no connections | Startup loads succeed with an empty map. Connect buttons fail. Chat integration tools report unconfigured |
| Push notifications | `OMI_DISABLED_ROUTES=POST /v1/users/fcm-token`. `firebase_messaging` is removed from the app and `NotificationService` becomes local-only (see "Client: Firebase retirement") | 404. With no stored tokens, `utils/notifications.py:246` `_send_to_user` sends nothing. `PUT /v1/users/time-zone` on the same router stays mounted | No push. Local notifications still show |
| Offline and SD-card sync upload | Client: `--dart-define=OMI_OFFLINE_AUDIO_UPLOAD=false` in the fork build (see "Client: offline audio upload switch"). Server backstop: `OMI_DISABLED_ROUTES=POST /v2/sync-capture-manifest;POST /v1/sync-local-files;POST /v2/sync-local-files;POST /v2/sync-local-files/{job_id}/release-hold;GET /v2/sync-local-files/{job_id}` | 404, but the client switch means no automatic caller reaches it | Recordings the live session did not cover stay on the phone, listed as not uploaded, with no upload attempts and no stalled-upload alert. Phase 2 path is Transcribe batch from S3 |
| Voice messages | `OMI_DISABLED_ROUTES=POST /v2/voice-messages;POST /v2/voice-message/transcribe;WS /v2/voice-message/transcribe-stream` | 404, and the WebSocket upgrade is refused | `voice_recorder_provider.dart` catches the failure |
| TTS | `OMI_DISABLED_ROUTERS=tts` | 404 on `/v2/tts/*` | Playback falls back to the system voice |
| Speech-profile enrolment | `OMI_DISABLED_ROUTES=GET /v4/speech-profile;POST /v3/upload-audio;GET /v3/speech-profile/expand;DELETE /v3/speech-profile/expand;GET /v3/speech-profile/stt-availability` | 404 | `GET /v3/speech-profile` stays mounted and returns `false`. The voice-profile flow's preflight reads the 404 as unavailable and shows `transcription_unavailable` |
| Review and dream | `OMI_DISABLED_ROUTERS=review,dream_cohort,dream_report` | 404 on `/v1/review/*`, `/v2/review/*`, `/v1/entities*`, `GET /v1/conversations/{conversation_id}/entities`, `PUT /v1/users/release-channel`, `/v1/dream/runs` | `ReviewProvider` reads 404 or 403 as `ReviewAvailability.off` (`app/lib/providers/review_provider.dart:172-176`). The Activity page treats a 404 from `/v1/review/changes` as no changes (`activity_page.dart:192-193`) |
| Google Maps static map | `OMI_DISABLED_ROUTERS=static_map`, plus `GOOGLE_MAPS_API_KEY` unset | 404 on `GET /v1/static-map` | `OmiMapPreview` falls back to its pin-dot canvas (`app/lib/widgets/omi_map_preview.dart:167-175`) |
| Frame and photo evidence | `OMI_DISABLED_ROUTERS=frame_requests`, plus `BUCKET_FRAME_REQUESTS*` unset | 404 on `/v1/frame-requests*` and `GET /v1/conversations/{conversation_id}/photos/{photo_id}/image` | The necklace has no camera. The photo cache returns null and shows a placeholder |
| Screenshots | `OMI_DISABLED_ROUTERS=screen_frames`, plus `BUCKET_SCREEN_FRAMES` unset | 404 on `/v1/conversations/{conversation_id}/screenshots*`, `/v1/conversations/{conversation_id}/shared/screenshots`, `/v1/screen-frame-egress/*` | The detail screenshot section hides on `ApiFailure` (`conversation_screenshots_section.dart:132-135`) |
| Owner voiceprint | `HOSTED_SPEAKER_EMBEDDING_API_URL` unset (unconfigured, no route) | `utils/stt/speaker_embedding.py:59` returns no embedding | Lower diarization quality |
| Gateway-only chat routes the app never calls: desktop chat completions (`routers/desktop_chat.py:1969`, `POST /v2/chat/completions`) and the web share page's chat (`routers/public_shared_conversation_chat.py:255`, `POST /v1/conversations/shared/chat`) | `OMI_DISABLED_ROUTERS=desktop_chat,public_shared_conversation_chat` | 404 | None. The Flutter app calls neither (`git grep -n "chat/completions\|shared/chat" app/lib` is empty). Both send every request to the LLM gateway, which the port does not run (see "Direct gateway callsites") |
| Perplexity web search tool | New `OMI_WEB_SEARCH_TOOL=off`, read where `utils/retrieval/agentic.py:1694` registers the tool. `PERPLEXITY_API_KEY` unset | The tool is not registered, so the model never sees it | Chat answers without web search. Today the tool stays registered and returns an error string |
| PostHog | `POSTHOG_PRODUCT_CAPTURE_TOKEN`, `POSTHOG_PROJECT_API_KEY`, `POSTHOG_API_KEY` and `POSTHOG_HOST` unset on the backend (`utils/product_telemetry.py:53-81`). The app guards on `Env.posthogApiKey != null` | No events sent | None |
| LangSmith | `LANGSMITH_TRACING=false`, no `LANGSMITH_API_KEY` (`utils/observability/langsmith.py:26`) | No tracing | None |
| Crashlytics and Intercom (client) | Crashlytics is removed from the fork's Flutter build and replaced by the local `CrashReporter` sink (see "Client: Firebase retirement"). Intercom is off when `Env.intercomAppId` is unset | n/a | Crash reports stay on the device. No support chat |
| Custom vocabulary | Not sent to Transcribe | n/a | Per-user keywords ignored |

The exact route spellings above are checked at startup, so the cut card can
copy them into the fork's runtime env file and get a startup failure if any
path has drifted.

**Caller retry policy.** A cut is acceptable only when no caller retries it on
a timer or loops on it. Data retention alone is not enough. Every cut route the
app can reach was checked for its caller's retry behaviour. The census lists
the result per route in its
[cut table](MVP-API-SURFACE.md#cut-routes-and-their-switches). Offline audio
upload is the only cut whose caller retries automatically, and the client
switch below removes that caller.

### Client: offline audio upload switch

The app uploads recordings the live session did not cover: audio captured
while the phone was offline or the socket was down, and SD-card audio
downloaded from the necklace. Without a client change, cutting the upload
routes leaves it retrying a 404 indefinitely:

- `autoSyncOfflineRecordings` defaults to true (`app/lib/backend/preferences.dart:741`),
  and `SyncProvider` passes it to `RecordingTransferCoordinator` as
  `autoUploadEnabled` (`app/lib/providers/sync_provider.dart:499-500`).
- A 404 is not a definitive refusal. Only 400 and 413 are
  (`app/lib/services/wals/wal_policy.dart:113`). The batch upload counts the
  404 as failed and leaves each WAL in `miss`
  (`app/lib/services/wals/local_wal_sync.dart:2337-2370`).
- The coordinator then retries after 5, 10, 20 and 60 seconds, and again on
  every ten-minute watchdog wake
  (`app/lib/services/wals/recording_transfer_coordinator.dart:154-157`, `:300-305`,
  `:504`, `:532-534`). After two hours the Activity page shows an "Upload stalled" row
  that never clears (`sync_provider.dart:678-683`, `:705-721`).

A server-side terminal refusal (answering 400 instead of 404) was rejected. The
client would park each WAL as `uploadRejected`, retry the refusal daily for
about seven days, upload the full audio on every attempt, and tell the user
the server refused their recording.

**Decision.** Add a compile-time define, `OMI_OFFLINE_AUDIO_UPLOAD`, read as
`const bool.fromEnvironment('OMI_OFFLINE_AUDIO_UPLOAD', defaultValue: true)`,
the same pattern as `mobileReferralCampaignEnabled`
(`app/lib/backend/http/api/referrals.dart:11`). The default keeps upstream
behaviour. The fork's Flutter build passes `false`. When it is false:

| Seam (paths under `app/lib/`) | Change |
| --- | --- |
| `backend/preferences.dart:741` `autoSyncOfflineRecordings` getter | Returns false regardless of the stored preference. This one change turns off every automatic caller: the coordinator's `autoUploadEnabled` (`providers/sync_provider.dart:500`), its watchdog (`services/wals/recording_transfer_coordinator.dart:300`) and its `mayUpload` check (504), Home's SD-card auto-sync on device connect (`pages/home/page.dart:674`), local-preview admission (`services/live_preview/local_preview_admission.dart:84`), and the capture controller (`services/capture/capture_controller.dart:2313`, `:2379`) |
| `pages/settings/device_settings.dart:546-556` | Hide the Auto Sync toggle, because it can no longer turn anything on |
| `providers/sync_provider.dart` `syncWals` (837), `syncWal` (861), and the retry paths at 871 and 1113; `providers/local_recordings_provider.dart:187` | Return without waking the coordinator. These are the manual `WakeTrigger.userRetry` entry points, which bypass `mayUpload`. Guarding them in the provider covers every button that calls them. Their buttons in Offline Sync are hidden |
| `providers/sync_provider.dart:678` `_isUploadStallCandidate` | Returns false, so `observeWalBacklog` (718) never raises the "Upload stalled" row for audio that is deliberately not uploaded |

**What the user sees.** Audio that the live session transcribed is confirmed
by its conversation and leaves the backlog as usual. Audio the live session
did not cover stays on the phone in `miss` and is listed in Offline Sync as
not uploaded, with no sync control and no stalled-upload alert. SD-card audio
stays on the necklace, because the device-connect auto-sync no longer starts
(`pages/home/page.dart:674`). WALs are never evicted for count: reaching
`maxRetainedCaptureWalCount` (720, `services/wals/wal_policy.dart:14`) only
raises the retention warning. New audio is refused when free space falls below
`minFreeDiskReserveBytes` (512 MiB, `wal_policy.dart:16`). The user can delete kept recordings from Offline Sync.

**Recovery.** Phase 2 restores the upload routes on Transcribe batch and ships
a build with the define on. Kept WALs then upload through the normal
coordinator path. Recordings older than `SYNC_BACKFILL_MAX_AGE_SECONDS`
(default 30 days, `backend/utils/sync/lanes.py:52`) are refused with 422
`backfill_lookback_exceeded` (`backend/routers/sync.py:790-797`), so phase 2
must either raise that setting or accept that older audio stays local.

The server cut stays as a backstop for older fork builds, which would retry as
described above until updated.

**Full switch values for the fork:**

```text
OMI_DISABLED_ROUTERS=auth,oauth,payment,retention_offer,yearly_switch,redeem_code,storefront,apps,developer,developer_key,phone_calls,messaging,tts,google_calendar,calendar_onboarding,calendar_meetings,x_connector,review,dream_cohort,dream_report,static_map,frame_requests,screen_frames,desktop_chat,public_shared_conversation_chat
OMI_DISABLED_ROUTES=GET /authorize;POST /authorize;POST /v1/users/fcm-token;POST /v1/conversations/search;POST /v2/sync-capture-manifest;POST /v1/sync-local-files;POST /v2/sync-local-files;POST /v2/sync-local-files/{job_id}/release-hold;GET /v2/sync-local-files/{job_id};POST /v2/voice-messages;POST /v2/voice-message/transcribe;WS /v2/voice-message/transcribe-stream;GET /v4/speech-profile;POST /v3/upload-audio;GET /v3/speech-profile/expand;DELETE /v3/speech-profile/expand;GET /v3/speech-profile/stt-availability;GET /v1/task-integrations/{app_key}/oauth-url;GET /v1/integrations/{app_key}/oauth-url;POST /v1/conversations/{conversation_id}/calendar-event;DELETE /v1/conversations/{conversation_id}/calendar-event;POST /v1/conversations/{conversation_id}/calendar-event/auto-link
```

Every module name is a `Mount(...)` in `ROUTER_MOUNTS`, and every route entry
matches its decorator exactly. A router switch removes every mount of that
module, so `developer` also removes nothing else that the MVP needs. The
v1 `POST /v1/sync-local-files` (`routers/sync.py:605`) is listed beside the v2
upload because both reach the cut batch STT path.

**Duplicate decorator.** `DELETE /v1/conversations/{conversation_id}/action-items`
is decorated at `routers/conversations.py:1820` and at `routers/action_items.py:1013`.
`conversations` mounts first (`app_roles.py:196`, before 202), so its handler
serves the route. The route switch matches by method and path, so a future
entry for this path would remove both. No cut needs it.

**Verification.** The cut-switch card adds a unit test beside
`tests/unit/test_app_roles_route_census.py`. It builds the app with the two
values above and asserts:

1. Every route in the MVP census's
   [cut table](MVP-API-SURFACE.md#cut-routes-and-their-switches) is absent from
   `app.router.routes`.
2. Every route the census marks **in** is still present.
3. No route whose path starts with a cut prefix (`/v1/auth/`, `/v1/oauth/`, `/v1/payments/`,
   `/v1/stripe/`, `/v1/apps`, `/v2/apps`, `/v1/phone/`, `/v1/messaging/`,
   `/v2/tts/`, `/v1/calendar/`, `/v1/x/`, `/v1/review/`, `/v2/review/`,
   `/v1/entities`, `/v1/dream/`, `/v1/frame-requests`, `/v1/static-map`)
   survives, except the two that are deliberately kept elsewhere:
   `POST /v1/tts/synthesize` (`routers/desktop_tts_updates.py:190`, desktop
   only, never called by the app) and the retired
   `GET /v1/frame-requests/temporary/{request_id}/image`
   (`routers/retired_app_client_contracts.py:107`, which answers 410).

The census lists are a data file the test reads, so a route marked cut in the
document cannot stay mounted unnoticed.

The client card adds a test in `app/test/services/wals/recording_transfer_coordinator_test.dart`
and `app/test/providers/sync_provider_sync_wal_wake_test.dart`, run with
`--dart-define=OMI_OFFLINE_AUDIO_UPLOAD=false`. It asserts that a pending WAL
produces no upload on watchdog, connectivity, cooldown or `userRetry` wakes,
and that no "Upload stalled" row is raised after the silence threshold.

## Consequences

- **Easier.** One Postgres engine serves the document adapter, CellStore and the
  jobstore. Every MVP model call stays in AWS under one IAM role. Auth needs no
  Google project.
- **Harder.** The GAPIC adapter must reproduce Firestore query semantics:
  ordering, cursors, `in` and `array-contains` filters, transforms, and
  preconditions. It needs a conformance suite run against the offline SDK
  fixture before any route is switched. Firestore's 1 MiB document limit and
  ordering rules become adapter tests, not database guarantees.
- **Watch.** In-process finalization keeps each wake in the task that admitted
  it. Revisit SQS when the backend runs more than one task. Recovery latency
  after a restart is up to 6 minutes, or 26 for a job killed mid-run. Transcribe
  multi-language with diarization. The cross-region `AdminGetUser` call. The
  `omi://` scheme collision. Fargate memory headroom with two containers. Any
  new Redis constructor that bypasses `redis_tls_kwargs` (the guard test
  catches it) and any new storage helper that returns a bare object URL.
- **Upstream drift.** New upstream Firestore features reach the fork through
  the adapter. A feature the adapter does not implement fails loudly at the
  transport rather than silently.

## Implemented in

Nothing yet. Implementation cards under the omi-aws initiative cite this ADR
when they change the seams named above.
