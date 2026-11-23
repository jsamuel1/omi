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
| Soniox / Modulate streaming STT | Amazon Transcribe streaming over a raw SigV4 WebSocket on Python 3.11, as a new `transcribe` provider and `TranscribeStreamingSocket(STTSocket)` | Self-hosted Parakeet on GPU (GPU cost floor for one user) | The closed STT policy seams listed under "STT: seam contract" | No |
| OpenAI / OpenRouter / Anthropic LLMs | Amazon Bedrock through `langchain-aws` `ChatBedrockConverse`, `au.` inference profiles | Bedrock's OpenAI-compatible endpoint (serves gpt-oss only, no Claude) | `utils/llm/providers.py` (new `bedrock` provider), `utils/llm/clients.py::get_llm` (762), `utils/llm/model_config.py` (new `MODEL_QOS` profile) | No |
| OpenAI embeddings | Bedrock `amazon.titan-embed-text-v2:0` (1024-dim) via `BedrockEmbeddings` | Cohere Embed on Bedrock (no gain at this scale, one more model to approve) | `utils/llm/clients.py::_OpenAIEmbeddingsProxy` (356) | No |
| OpenAI Files (chat attachments) | S3 `BUCKET_CHAT_FILES` for the bytes, sent to Bedrock Converse as image and document content blocks | Keeping OpenAI Files (not an AWS service, and a second LLM vendor for one feature) | `utils/other/chat_file.py` `FileChatTool.upload` (249), `upload_video`, `_ask_files_stream` (397), `_ask_files` (462), `download_owned_chat_file` (115) | No |
| Gemini / ElevenLabs TTS (`/v2/tts/synthesize`) | Cut for MVP. Amazon Polly later | Polly now (the client already falls back to the system voice) | `routers/tts.py` | No |
| Pinecone (conversation vectors) | Cut for MVP. pgvector on the same Aurora cluster later | OpenSearch Serverless (OCU minimum is a standing cost for one user) | `database/vector_db.py` (`PINECONE_API_KEY`/`PINECONE_INDEX_NAME` at 144-155) | No |
| Typesense (conversation search) | Cut for MVP. Postgres full-text search later | OpenSearch managed domain (standing instance cost) | `utils/conversations/search.py::search_conversations` (342) | No |
| Google Maps Static API (`GET /v1/static-map`, `utils/static_map.py:343`, `GOOGLE_MAPS_API_KEY`) | Cut for MVP via `OMI_DISABLED_ROUTERS=static_map`. Amazon Location Service static maps later | Keeping the Google key (a GCP credential in an AWS-only port) | `routers/static_map.py`, `utils/static_map.py::fetch_static_map` | No |
| Firebase Cloud Messaging push | Cut for MVP via `OMI_DISABLED_ROUTES` (see "Cut switches"). SNS mobile push to APNs later | Amazon Pinpoint (end of support 2026-10-30) | `POST /v1/users/fcm-token` (`routers/notifications.py:35`), `utils/notifications.py` | No |
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

**Decision.** Widen the existing provider-neutral seam instead of adding a
second one. `AuthTokenGateway` (`services/auth/auth_token_result.dart:80`)
already exposes `currentUser` as an `AuthUserSnapshot` (60), `forceRefresh` as
a `RefreshedAuthToken` (73), and `signOut`. Its only implementation is
`_FirebaseAuthTokenGateway` (`services/auth_service.dart:27`). Rename it
`AuthSessionGateway` and add:

| Member | Contract |
| --- | --- |
| `AuthUserSnapshot? get currentUser` | Synchronous, from the in-memory session. `uid` is the Cognito `sub` |
| `Stream<AuthUserSnapshot?> userChanges()` | Emits on sign-in, sign-out and account switch. Replaces `authStateChanges()` |
| `Stream<RefreshedAuthToken> tokenChanges()` | Emits after each refresh. Replaces `idTokenChanges()` |
| `Future<RefreshedAuthToken?> forceRefresh()` | Refresh-token grant against `/oauth2/token` |
| `Future<AuthSignInResult> signIn()` | Hosted UI authorization code with PKCE. `AuthSignInResult` holds the snapshot and an `isNewUser` flag, replacing `UserCredential` |
| `Future<void> signOut()` | `/oauth2/revoke` on the refresh token, local wipe, then hosted-UI `/logout` |
| `Future<void> updateDisplayName(String)` | Local profile field only. The pool is admin-managed |

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
| `services/auth_service.dart` | 791 `signInWithLocalDevToken` | Deleted with the backend local-dev token route |
| `services/auth_service.dart` | 513-580 `refreshIdToken` | Becomes `forceRefresh()` |
| `services/auth_service.dart` | 993, 1024 | Display-name update. Becomes `updateDisplayName` |
| `providers/auth_provider.dart` | 50, 80, 99, 187, 190 | Signed-in and owner checks |
| `providers/auth_provider.dart` | 116 `authStateChanges`, 141 and 144 `idTokenChanges` | Become `userChanges()` and `tokenChanges()` |
| `providers/auth_provider.dart` | 236, 268, 311-343 | `UserCredential` sign-in flow. Becomes `AuthSignInResult` |
| `services/onboarding_sync_runtime.dart` | 34 | Auth listener that wakes onboarding sync |
| `services/on_device/on_device_background.dart` | 35, 49 | Owner check and auth listener for background work |
| `services/siri_integration.dart` | 1051 | Owner check before opening a Siri result |
| `services/telemetry/client_telemetry_transport.dart` | 18, 33 | Telemetry owner id and token |
| `pages/chat/widgets/content_blocks/edit_receipt_block.dart` | 253, 258 | Owner check for receipt undo |
| `pages/home/home_deep_links.dart` | 205, 208, 256, 260, 291, 293 | Owner checks on deep-link resolution |
| `pages/home/item_list_sheet.dart` | 60, 66 | Owner check |
| `pages/onboarding/wrapper.dart` | 627-629 | Email, name and uid for onboarding |
| `pages/onboarding/variants/yinka/yinka_onboarding_flow.dart` | 156-158 | Same |
| `pages/settings/change_name_widget.dart` | 20 | Firebase `User` field |
| `utils/auth/sign_in_failure.dart` | 32 | `FirebaseAuthException` mapping. Maps OAuth error codes instead |

**Removal list:** `startup_firebase.dart`, `firebase_options_local.dart`,
`utils/debugging/crashlytics_manager.dart`; `firebase_core`, `firebase_auth`,
`firebase_messaging`, `firebase_crashlytics`, `google_sign_in` and
`sign_in_with_apple` at `pubspec.yaml:40-47`, plus `google_sign_in_ios` (161)
and the Firebase test interfaces (178, 197); the Google services and
Crashlytics Gradle plugins at `android/settings.gradle:23-24` and
`android/app/build.gradle:4-5`; `google-services.json` and
`GoogleService-Info.plist` with their Xcode project references. Push moves to a
no-op `NotificationService`.

**Acceptance check for the auth card:** `grep -r 'package:firebase_' app/lib`
returns nothing, and sign-in, cold restore, sign-out and account switch work
against the pool.

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
- **Agent loop fit.** `utils/retrieval/agentic.py::_run_openai_agent_stream`
  calls `get_llm('chat_agent')` and reads `tool_call_chunks`, which
  `ChatBedrockConverse` emits. Strip OpenAI-only kwargs (for example
  `stream_options`) in the provider. The `_AnthropicClientProxy` path
  (`agentic.py` around line 963) switches to `anthropic.AsyncAnthropicBedrock`.
- **Gateway.** `OMI_LLM_GATEWAY_FEATURE_MODE` stays unset, so features call the
  model directly.

### Chat files: S3 plus Bedrock content blocks

`POST /v2/files` and `/v2/files/video` upload attachments to OpenAI Files today
(`utils/other/chat_file.py:249` and 343), and the answer path calls OpenAI chat
completions directly (`_ask_files_stream` at 397, `_ask_files` at 462).

- Store the bytes in `BUCKET_CHAT_FILES` under `{uid}/{file_id}`. Keep the
  `openai_file_id` field on `u/files/{id}` and put the S3 key in it, so the
  rows and `download_owned_chat_file` (115) keep their shape.
- At answer time, read the object and send it to `ChatBedrockConverse` as an
  image or document content block, replacing both `_ask_files*` paths.
- Cleanup deletes the S3 object instead of the OpenAI file.
- The thumbnail bucket write (`utils/other/storage.py:2343`) is unchanged apart
  from the storage client.

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

**Pusher sidecar.** `schedule_finalization`
(`routers/listen/conversations.py:330`) logs "Pusher unavailable; finalization
remains queued" and returns `False` when no pusher client exists and
listen-finalization dispatch is off. Conversations would never finalize. The
pusher image (`backend/pusher/Dockerfile`) therefore runs in the same task,
with `HOSTED_PUSHER_API_URL=http://localhost:8081`. It stays off the ALB,
because `/v1/trigger/listen` takes a `uid` query parameter without
authenticating it.

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
| `BUCKET_CHAT_FILES` | `omi-chat-files-717494614307-apse2` | chat attachments |
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
- `OMI_ROLE` unset, so `app_roles.py` mounts every router role, then prunes the
  cut routers and routes named by `OMI_DISABLED_ROUTERS` and
  `OMI_DISABLED_ROUTES` (see "Cut switches").
- `DIRECTORY_ROUTING_MODE` unset (off), `FAIR_USE_ENABLED=false`,
  `STT_CAP_DISPATCH_ENFORCEMENT=false`.
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
| Firebase sign-in (`/v1/auth/*`) | `OMI_DISABLED_ROUTERS=auth` | 404 on `/v1/auth/authorize`, `/callback/*`, `/token`, `/local-dev/custom-token` | Never called. The app uses the Cognito hosted UI. `oauth` stays mounted for MCP |
| Stripe billing | `OMI_DISABLED_ROUTERS=payment,retention_offer,yearly_switch,redeem_code,storefront`, plus `STRIPE_API_KEY` and `STRIPE_*_PRICE_ID` unset | 404 on `/v1/payments/*`, `/v1/stripe/*`, `/v1/payment-methods/*` | `GET /v1/users/me/subscription` stays mounted and returns the seeded `unlimited` plan without calling Stripe. `showSubscriptionUI` is false, so the paywall never opens |
| Plugins / app store | `OMI_DISABLED_ROUTERS=apps,developer,developer_key` | 404 on `/v1/apps*`, `/v2/apps*` | Conversation detail keeps its cached summary templates (`SummaryTemplatesLoad.failed`) |
| Typesense search | `OMI_DISABLED_ROUTES=POST /v1/conversations/search`, plus `TYPESENSE_HOST` and `TYPESENSE_API_KEY` unset | 404 | Search returns `ConversationSearchResult.failure` (`app/lib/backend/http/api/conversations.dart:1312`). With the env vars unset, `utils/conversations/typesense_index.py` skips index writes |
| Pinecone | `PINECONE_API_KEY`, `PINECONE_INDEX_NAME` unset (unconfigured, no route) | Index is `None`. Upserts return `False`, queries return empty | Chat retrieval skips the vector branch (`utils/retrieval/tool_services/conversations.py:330`) |
| Twilio phone calls | `OMI_DISABLED_ROUTERS=phone_calls`, plus `TWILIO_*` unset | 404 on `/v1/phone/*` | `/v4/listen?source=phone_call` cannot start, because the client needs `POST /v1/phone/token` first |
| Telegram / iMessage messaging | `OMI_DISABLED_ROUTERS=messaging`, plus `OMI_TELEGRAM_ENABLED=off` and `OMI_IMESSAGE_ENABLED=off` (the defaults, `utils/messaging/adapters/runtime.py:23,35`) | 404 on `/v1/messaging/*`. `start_adapters` returns `None` | Channel chats page unreachable |
| Google Calendar and X | `OMI_DISABLED_ROUTERS=google_calendar,calendar_onboarding,calendar_meetings,x_connector`, plus `OMI_DISABLED_ROUTES=POST /v1/conversations/{conversation_id}/calendar-event;DELETE /v1/conversations/{conversation_id}/calendar-event;POST /v1/conversations/{conversation_id}/calendar-event/auto-link` | 404 on `/v1/calendar/*` (including `/v1/calendar/meetings*`, a separate module), `/v1/x/*`, and the three conversation calendar-event routes, which live in the retained `conversations` module | Auto-link returns null on any failure. "Link event" asks to connect Google Calendar, which never connects |
| Task and app integrations (Google Tasks, Todoist, Asana, ClickUp, Gmail) | `OMI_DISABLED_ROUTES=GET /v1/task-integrations/{app_key}/oauth-url;GET /v1/integrations/{app_key}/oauth-url`, plus provider client ids unset | 404 on the OAuth starts. The status reads stay mounted and return no connections | Startup loads succeed with an empty map. Connect buttons fail. Chat integration tools report unconfigured |
| Push notifications | `OMI_DISABLED_ROUTES=POST /v1/users/fcm-token`. `firebase_messaging` is removed from the app and `NotificationService` becomes a no-op | 404. With no stored tokens, `utils/notifications.py:246` `_send_to_user` sends nothing. `PUT /v1/users/time-zone` on the same router stays mounted | No push |
| Offline and SD-card sync upload | `OMI_DISABLED_ROUTES=POST /v2/sync-capture-manifest;POST /v2/sync-local-files;POST /v2/sync-local-files/{job_id}/release-hold;GET /v2/sync-local-files/{job_id}` | 404 | The WAL treats only 400 and 413 as definitive refusals (`app/lib/services/wals/wal_policy.dart:113`), so a 404 keeps audio on the device. Phase 2 path is Transcribe batch from S3 |
| Voice messages | `OMI_DISABLED_ROUTES=POST /v2/voice-messages;POST /v2/voice-message/transcribe;WS /v2/voice-message/transcribe-stream` | 404, and the WebSocket upgrade is refused | `voice_recorder_provider.dart` catches the failure |
| TTS | `OMI_DISABLED_ROUTERS=tts` | 404 on `/v2/tts/*` | Playback falls back to the system voice |
| Speech-profile enrolment | `OMI_DISABLED_ROUTES=GET /v4/speech-profile;POST /v3/upload-audio;GET /v3/speech-profile/expand;DELETE /v3/speech-profile/expand;GET /v3/speech-profile/stt-availability` | 404 | `GET /v3/speech-profile` stays mounted and returns `false`. The voice-profile flow's preflight reads the 404 as unavailable and shows `transcription_unavailable` |
| Review and dream | `OMI_DISABLED_ROUTERS=review,dream_cohort,dream_report` | 404 on `/v1/review/*`, `/v2/review/*`, `/v1/entities*`, `GET /v1/conversations/{conversation_id}/entities`, `PUT /v1/users/release-channel`, `/v1/dream/runs` | `ReviewProvider` reads 404 or 403 as `ReviewAvailability.off` (`app/lib/providers/review_provider.dart:172-176`). The Activity page treats a 404 from `/v1/review/changes` as no changes (`activity_page.dart:192-193`) |
| Google Maps static map | `OMI_DISABLED_ROUTERS=static_map`, plus `GOOGLE_MAPS_API_KEY` unset | 404 on `GET /v1/static-map` | `OmiMapPreview` falls back to its pin-dot canvas (`app/lib/widgets/omi_map_preview.dart:167-175`) |
| Frame and photo evidence | `OMI_DISABLED_ROUTERS=frame_requests`, plus `BUCKET_FRAME_REQUESTS*` unset | 404 on `/v1/frame-requests*` and `GET /v1/conversations/{conversation_id}/photos/{photo_id}/image` | The necklace has no camera. The photo cache returns null and shows a placeholder |
| Screenshots | `OMI_DISABLED_ROUTERS=screen_frames`, plus `BUCKET_SCREEN_FRAMES` unset | 404 on `/v1/conversations/{conversation_id}/screenshots*`, `/v1/conversations/{conversation_id}/shared/screenshots`, `/v1/screen-frame-egress/*` | The detail screenshot section hides on `ApiFailure` (`conversation_screenshots_section.dart:132-135`) |
| Owner voiceprint | `HOSTED_SPEAKER_EMBEDDING_API_URL` unset (unconfigured, no route) | `utils/stt/speaker_embedding.py:59` returns no embedding | Lower diarization quality |
| Perplexity web search tool | New `OMI_WEB_SEARCH_TOOL=off`, read where `utils/retrieval/agentic.py:1694` registers the tool. `PERPLEXITY_API_KEY` unset | The tool is not registered, so the model never sees it | Chat answers without web search. Today the tool stays registered and returns an error string |
| PostHog | `POSTHOG_PRODUCT_CAPTURE_TOKEN`, `POSTHOG_PROJECT_API_KEY`, `POSTHOG_API_KEY` and `POSTHOG_HOST` unset on the backend (`utils/product_telemetry.py:53-81`). The app guards on `Env.posthogApiKey != null` | No events sent | None |
| LangSmith | `LANGSMITH_TRACING=false`, no `LANGSMITH_API_KEY` (`utils/observability/langsmith.py:26`) | No tracing | None |
| Crashlytics and Intercom (client) | Removed from the fork's Flutter build. Intercom is also off when `Env.intercomAppId` is unset | n/a | No crash reports or support chat |
| Custom vocabulary | Not sent to Transcribe | n/a | Per-user keywords ignored |

The exact route spellings above are checked at startup, so the cut card can
copy them into the fork's runtime env file and get a startup failure if any
path has drifted.

**Full switch values for the fork:**

```text
OMI_DISABLED_ROUTERS=auth,payment,retention_offer,yearly_switch,redeem_code,storefront,apps,developer,developer_key,phone_calls,messaging,tts,google_calendar,calendar_onboarding,calendar_meetings,x_connector,review,dream_cohort,dream_report,static_map,frame_requests,screen_frames
OMI_DISABLED_ROUTES=POST /v1/users/fcm-token;POST /v1/conversations/search;POST /v2/sync-capture-manifest;POST /v1/sync-local-files;POST /v2/sync-local-files;POST /v2/sync-local-files/{job_id}/release-hold;GET /v2/sync-local-files/{job_id};POST /v2/voice-messages;POST /v2/voice-message/transcribe;WS /v2/voice-message/transcribe-stream;GET /v4/speech-profile;POST /v3/upload-audio;GET /v3/speech-profile/expand;DELETE /v3/speech-profile/expand;GET /v3/speech-profile/stt-availability;GET /v1/task-integrations/{app_key}/oauth-url;GET /v1/integrations/{app_key}/oauth-url;POST /v1/conversations/{conversation_id}/calendar-event;DELETE /v1/conversations/{conversation_id}/calendar-event;POST /v1/conversations/{conversation_id}/calendar-event/auto-link
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
3. No route whose path starts with a cut prefix (`/v1/auth/`, `/v1/payments/`,
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
