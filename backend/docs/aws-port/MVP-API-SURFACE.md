# MVP API surface for the AWS port

Companion to [ADR-001](ADR-001-aws-service-mapping.md). This census lists every
backend route the Flutter app (`app/`) calls for the MVP loop: sign-in and
bootstrap, device and firmware, live capture, conversations, memories and chat.
Each row gives the client caller, the backend handler, what the handler touches,
and whether the route is in the MVP.

Line references are to `origin/main` at `a04153d180`. Client paths are relative
to `app/lib/`, and backend paths to `backend/`. A route is "in" when the MVP
port must serve it correctly. It is "cut" when its feature is disabled by a
switch in ADR-001's [cut switches](ADR-001-aws-service-mapping.md#cut-switches)
and the client keeps working without it.

## How the census was built

1. **Reach.** Start from the signed-in startup path
   (`core/app_shell.dart:420` `_initializeProviders`, `pages/home/page.dart`)
   and the screens of the six MVP areas: Home and the conversation list,
   conversation detail, capture, memories, chat, and device settings.
2. **Callers.** For every provider and widget reached from those screens, follow
   each call into `backend/http/api/*.dart` and the socket clients under
   `services/sockets/`, and record the URL.
3. **Handlers.** Match the URL to its `@router` decorator in `backend/routers/`,
   then follow the handler to the collections, buckets and providers it
   touches.
4. **Classify.** A route reached automatically on a signed-in startup, or by a
   primary control in an MVP screen, is "in". It moves to "cut" only when its
   feature is cut and the client was checked to tolerate the failure. Routes
   whose only callers sit behind a cut feature or a non-MVP page are listed
   under [Checked, not reached](#checked-not-reached).

## How to read the chains

**Store** names Firestore collections. Under `users/{uid}/...` is written as
`u/…`. After the port, every collection is a path prefix in the Postgres
`documents` table behind the Firestore adapter (ADR-001, "Document store").
GCS buckets become the S3 buckets in ADR-001's bucket table. **Providers** are
external services: after the port, LLM means Bedrock, STT means Transcribe,
and Vector means Pinecone, which is cut so calls return empty.

### Shared auth chain (every row)

Every HTTP route resolves the uid through
`utils/other/endpoints.py::get_current_user_uid` (271), or `_readonly` (257).
Both call `verify_token` (143). Many add `with_rate_limit` (733), which keeps
counters in Redis. The WebSocket uses `get_current_user_uid_ws_listen` (462),
which goes through `_verify_ws_auth` (390), checks the account-deletion marker
(`account_deletions/{uid}`) and validates BYOK headers.

| Today | After the port |
| --- | --- |
| Firebase ID token, `firebase_admin.auth.verify_id_token` | Cognito access token from pool `ap-southeast-4_jA7oAYYrp`, verified with PyJWT against the pool JWKS, uid = `sub` |
| Client gets the token from `FirebaseAuth.currentUser.getIdToken` (`services/auth_service.dart`) | Client gets it from the hosted UI PKCE exchange at `https://auth.home.sauhsoj.wtf/oauth2/token` and refreshes with the refresh token |
| Header `Authorization: Bearer <token>` set in `backend/http/shared.dart:263` | Unchanged |

Rate-limit and deletion-marker reads hit Redis (`REDIS_DB_HOST`, now Valkey) and
`account_deletions`. Those two dependencies apply to every row and are not
repeated below.

## 1. Sign-in and bootstrap

There is no backend sign-in route in the MVP. The app signs in against the
Cognito hosted UI directly. `/v1/auth/*` (`routers/auth.py`, the Firebase
custom-token flow for Google and Apple) is cut.

**Startup order.** `_initializeProviders` (`core/app_shell.dart:420`) first
awaits `GET /v1/users/deletion-status`. It returns `false`, and startup stops,
unless that call is an `ApiSuccess` with no pending deletion (line 429). It then
starts, without awaiting, the speech-profile check, the language list, the
user and people providers, the message refresh, the subscription fetch, and the
task-integration and integration loads (lines 431-451).

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `GET /v1/users/deletion-status` | `backend/http/api/account_deletion.dart` `status()`, via `providers/auth_provider.dart:79` `checkAccountDeletion`, awaited at `core/app_shell.dart:427` | `routers/account_deletion.py:76`, then `services/users/account_deletion.py:769` | `account_deletions/{uid}` marker (`database/account_deletion_marker.py:95`), the held-audio summary, and the processing backlog (`database/processing_backlog.py:43`) | in. Startup aborts on failure |
| `GET /v1/users/available-languages` | `users.dart:519`, via `providers/home_provider.dart:257` `loadLanguagesThenSetupPrimary`, called at `app_shell.dart:434` | `routers/users.py:924` | None. Returns the static `PRIMARY_LANGUAGE_OPTIONS` | in |
| `GET /v1/users/private-cloud-sync` | `users.dart:328`, via `providers/user_provider.dart:178` `initialize`, called at `app_shell.dart:435` | `routers/users.py:652` | `users/{uid}` (private-cloud-sync flag) | in |
| `GET /v1/users/training-data-opt-in` | `users.dart:628`, via `user_provider.dart:178` | `routers/users.py:1151` | `users/{uid}` (opt-in field) | in |
| `GET /v1/users/people` | `users.dart:383`, via `PeopleProvider.initialize` (`app_shell.dart:436`) and `HomeProvider.setUserPeople` | `routers/users.py:705` | `u/people`, plus signed URLs for speech samples in S3 `BUCKET_SPEECH_PROFILES` (`utils/other/storage.py:453`) | in |
| `GET /v1/users/me/subscription` | `users.dart:695`, via `providers/usage_provider.dart` `fetchSubscription`, called at `app_shell.dart:447` and `pages/home/page.dart:131` | `routers/users.py:1283` | `users/{uid}.subscription`. The `is_marketplace_reviewer(uid)` branch (`users.py:1437`) returns before any Stripe call. Otherwise `reconcile_basic_plan_with_stripe` (`utils/subscription.py:1765`) calls Stripe only for a basic or lapsed plan that carries a `stripe_subscription_id`, which the seeded owner does not | in. The paywall bails when `showSubscriptionUI` is false (`home/page.dart:140`) |
| `GET /v1/task-integrations` | `backend/http/api/task_integrations.dart:29`, via `TaskIntegrationProvider.loadFromBackend` at `app_shell.dart:448` | `routers/task_integrations.py:233` | `u/task_integrations` (`database/users.py:2108`) | in. Non-200 sets a provider load error and the app continues |
| `GET /v1/integrations/{app_key}` | `backend/http/api/integrations.dart:29`, via `IntegrationProvider.loadFromBackend` at `app_shell.dart:451` | `routers/integrations.py:293` | `u/integrations/{app_key}` (`database/users.py:2241`) | in. Same tolerance as above |
| `PATCH /v1/users/geolocation` | `users.dart:169`, called from `pages/home/page.dart:450` and the capture path | `routers/users.py:444` | Redis geolocation cache only | in |
| `GET /v1/users/onboarding` | `backend/http/api/users.dart:900` `getUserOnboardingState`, called from `services/auth_service.dart:965` and `pages/home/page.dart:641` | `routers/users.py:606` | `users/{uid}` (onboarding fields) | in |
| `PATCH /v1/users/onboarding` | `users.dart:910` `updateUserOnboardingState`, called from `services/onboarding_sync_runtime.dart:18` | `routers/users.py:627` | `users/{uid}` (onboarding state, private-cloud-sync flag) | in |
| `GET /v1/users/language` | `users.dart:538` `getUserPrimaryLanguage`, called from `auth_service.dart:980` and `providers/home_provider.dart:271` | `routers/users.py:930` | `users/{uid}` (language) | in |
| `PATCH /v1/users/language` | `users.dart:559` `setUserPrimaryLanguage`, called from `home_provider.dart:317` | `routers/users.py:941` | `users/{uid}` (language, transcription preferences) | in |
| `GET /v1/users/profile` | `backend/http/api/privacy.dart:32`, called from `providers/user_provider.dart:187` | `routers/users.py:314` | `users/{uid}` | in |
| `GET /v1/users/transcription-preferences` | `users.dart:661` `getTranscriptionPreferences`, called from `user_provider.dart:60` | `routers/users.py:971` | `users/{uid}` (transcription preferences) | in |
| `PATCH /v1/users/transcription-preferences` | `users.dart:683` `setTranscriptionPreferences`, called from `user_provider.dart:61,322` | `routers/users.py:978` | `users/{uid}` (transcription preferences) | in |
| `PUT /v1/users/time-zone` | `backend/http/api/notifications.dart:9` `syncUserTimeZoneServer`, called from `providers/auth_provider.dart:348` | `routers/notifications.py:65` | `users/{uid}` via `notification_db.sync_user_time_zone_from_client` | in |
| `GET /v1/flags?surface=mobile` | `backend/http/api/flags.dart:13` `getClientFlags`, called from `utils/analytics/analytics_manager.dart:253` | `routers/omi_flags.py:28` | `omi_flags.resolve_surface`. The project flag store stays unset, so defaults apply | in |
| `GET /v3/speech-profile` | `backend/http/api/speech_profile.dart:18` `userHasSpeakerProfile`, called from `home_provider.dart:242` | `routers/speech_profile.py:62` | S3 `BUCKET_SPEECH_PROFILES` object `{uid}/speech_profile.wav` exists (`utils/other/storage.py:319`). Returns `false` when the bucket is unset | in |
| `POST /v1/users/client-state` | `services/sockets/listen_client_state.dart:104` | `routers/users.py:2793` | Redis sorted sets and keys under `demand-processing:v1:` (`utils/conversations/demand_processing.py::observe_surface`). The prefix needs `OMI_ENV_STAGE` set | in |
| `POST /v1/users/fcm-token` | `notifications.dart:18` `saveFcmTokenServer`, called only from the FCM service | `routers/notifications.py:35` | `u/fcm_tokens` | cut. Route disabled by `OMI_DISABLED_ROUTES`. The client `NotificationService` becomes a no-op, so the call at `app_shell.dart:453` sends nothing |

### Home

These run when Home and the conversation list mount.

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `GET /v1/action-items` | `backend/http/api/action_items.dart:78` `tryGetActionItems`, via `ActionItemsProvider.fetchActionItems` (411) and `ensureLoaded` (276), called from Home | `routers/action_items.py:582` | `u/action_items` | in |
| `GET /v1/pending-work` | `backend/http/api/pending_work.dart:20`, called from `pages/conversations/conversations_page.dart:399` | `routers/pending_work.py:33` | Rows from `database/processing_attention.py` and `database/sync_jobs.py` | in. The client treats 404 as empty |
| `GET /v1/attention-items` | `pending_work.dart:47`, same caller | `routers/pending_work.py:64` | Rows from `database/processing_attention.py` | in |
| `GET /v1/users/daily-summaries?limit=1` | `users.dart:787`, via `providers/conversation_provider.dart:696` `checkHasDailySummaries`, run by `getInitialConversations` (1107) from `conversations_page.dart:889` | `routers/users.py:2076` | Daily summaries under `users/{uid}` (`database/daily_summaries.py`) | in. Empty list on a fresh account |
| `GET /v1/episodes/live` | `backend/http/api/episodes.dart:30`, via `EpisodeProvider.refreshLive` (516) | `routers/episodes.py:85` | Episode admission config. Returns a disabled snapshot unless the client channel is admitted | in |
| `GET /v1/announcements/pending` | `backend/http/api/announcements.dart:50`, via `services/announcement_service.dart:61`, triggered at `home/page.dart:603` | `routers/announcements.py:119` | `announcements`, `u/dismissed_announcements` (`database/announcements.py:479`) | in |
| `GET /v1/announcements/changelogs` | `announcements.dart:12`, via `announcement_service.dart:81` | `routers/announcements.py:35` | `announcements` | in |

## 2. Device and firmware

Pairing and audio over BLE are on-device and need no backend route. The backend
serves only the firmware manifest.

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `GET /v2/firmware/latest` | `backend/http/api/device.dart:16` `getLatestFirmwareVersion`, called from `pages/home/firmware_mixin.dart:396` and `providers/device_provider.dart:1105` | `routers/firmware.py:360` | GitHub Releases API for `BasedHardware/omi` (`utils/github_releases.py`), cached in Redis with a last-known-good key. Optional `GITHUB_TOKEN` | in |
| `GET /v2/firmware/stable?device_model=…` | `device.dart:56` `getStableFirmwareVersion`, called from `firmware_mixin.dart:412` and `device_provider.dart:1504` | `routers/firmware.py:408` | Same as above | in |

## 3. Live capture

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `WS /v4/listen` | `services/sockets/transcription_service.dart:184` | `routers/transcribe.py:191` `listen_handler`, then `routers/listen/` (receiver, runtime, transcripts, conversations) | See the chain below | in |
| `PATCH /v1/conversations/{id}/segments/assign-bulk` | `backend/http/api/conversations.dart:753` `assignBulkConversationTranscriptSegments`, called from `pages/conversation_detail/widgets/transcript_tab.dart:300` and `conversation_detail_provider.dart:70` | `routers/conversations.py:1990`, then `_assign_manual_speaker` (1858) | `u/conversations/{id}` (transcript segments, in a transaction), `u/people` | in |

**`/v4/listen` query contract.** `language` (default `en`), `sample_rate`
(default 8000), `codec` (default `pcm8`), `channels`, `include_speech_profile`,
`stt_service`, `conversation_timeout` (default 120 s), `source`,
`client_conversation_id`, `conversation_role`, `speaker_auto_assign`,
`vad_gate`, `local_preview`, and others listed at `routers/transcribe.py:192-214`.
The client sends binary audio frames and pings every 20 s. The server sends
JSON segment and event messages.

**`/v4/listen` dependency chain:**

1. **Auth.** `get_current_user_uid_ws_listen`. Close 4001 means refresh the
   token, 1008 means rejected.
2. **Session bootstrap.** Reads `users/{uid}` (language, preferences,
   subscription) and `u/fair_use_state`, and writes `u/recording_sessions` and
   `u/client_devices`.
3. **Decode.** `routers/listen/receiver.py:930-965` turns opus, aac, lc3 or pcm8
   into PCM16 in process.
4. **STT.** `utils/stt/streaming.py` picks a provider from `STT_SERVICE_MODELS`.
   Today that is Modulate then Soniox. After the port it is Amazon Transcribe
   streaming through `TranscribeStreamingSocket`, forwarding finals only, with
   speakers mapped to `SPEAKER_NN`.
5. **Transcript persistence.** `u/conversations/{id}` holds the in-progress
   conversation and its segments.
6. **Audio archive.** S3 `BUCKET_PRIVATE_CLOUD_SYNC` and `BUCKET_CANONICAL_AUDIO`.
   Private cloud sync defaults on.
7. **Usage.** `u/hourly_usage` and `u/llm_usage`. The transcription allowance
   is bypassed for `MARKETPLACE_APP_REVIEWERS`.
8. **Finalization.** After `conversation_timeout` seconds of silence,
   `routers/listen/conversations.py:330` `schedule_finalization` calls
   `lifecycle_service.request_finalization`, which writes
   `conversation_finalization_jobs` and
   `conversation_finalization_projection_shards`. It then posts to the pusher
   sidecar (`HOSTED_PUSHER_API_URL=http://localhost:8081`). Without pusher the
   job stays queued and the conversation never finalizes.
9. **Processing on the pusher side.** `utils/conversations/process_conversation.py::process_conversation`
   (2585) runs the LLM features `conv_structure`, `conv_discard`, `conv_folder`,
   `conv_app_select`, `conv_app_result`, `memories` (three passes),
   `memory_category` and `learnings`. It writes `u/conversations/{id}`
   (structured fields), `u/memory_items` and `u/memory_operations`. The vector
   upsert goes to Pinecone, which is cut and returns `False`. Push is cut, so it
   sends to zero tokens.

## 4. Conversations

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `GET /v1/conversations?limit&offset&statuses&include_discarded` | `conversations.dart:513` `getConversationsResult`, called from `providers/conversation_home_fetch.dart:124` | `routers/conversations.py:963` `get_conversations` | `u/conversations`, ordered by `created_at`, via `conversations_db.get_conversations_without_photos` | in |
| `GET /v1/conversations/{id}` | `conversations.dart:588` `getConversationById`, called from `pages/conversation_detail/conversation_detail_provider.dart:87` and `widgets/summary_tab.dart:71` | `routers/conversations.py:1136` | `u/conversations/{id}` | in |
| `POST /v1/conversations` (process the in-progress conversation now) | `conversations.dart:48` `processInProgressConversationOutcome`, called from `services/capture/capture_controller.dart:4186` | `routers/conversations.py:503` | `lifecycle_service.admit_processing`, then the finalization chain of section 3 steps 8 and 9. Redis cached geolocation | in |
| `POST /v1/conversations/{id}/reprocess` | `conversations.dart:235` `reProcessConversationServer`, called from `conversation_detail_provider.dart:86` and `pages/activity/needs_attention_section.dart:277` | `routers/conversations.py:759` | `u/conversations/{id}`, then `process_conversation` (LLM features above, `u/memory_items`). Hourly limit in Redis | in |
| `PATCH /v1/conversations/{id}/title?title=…` | `conversations.dart:641` `updateConversationTitle`, called from `conversation_detail_provider.dart:1008` | `routers/conversations.py:1194` | `u/conversations/{id}` (`title`) | in |
| `PATCH /v1/conversations/{id}/starred?starred=…` | `conversations.dart:780` `setConversationStarred`, called from `pages/conversations/conversation_actions.dart:177` and `pages/conversation_detail/page.dart:837` | `routers/conversations.py:2218` | `u/conversations/{id}` (`starred`), `u/people`, Redis conversation-owner key | in |
| `DELETE /v1/conversations/{id}?cascade=true` | `conversations.dart:325` `deleteConversationServer`, called from `providers/conversation_provider.dart:1643` | `routers/conversations.py:1436` | `destructive_operation_gate` (legal-hold and deletion gate docs), `u/conversations/{id}` and its `photos` subcollection (`database/conversations.py:2594`), memory retraction in `u/memory_items` (`memory_service.retract_conversation_memories`), S3 audio prefixes (`utils/other/audio_packs.py:126`), Pinecone delete (cut, so a no-op) | in |
| `GET /v1/sync/audio/{id}/precache` | `backend/http/api/audio.dart:132`, called from `conversation_detail_provider.dart:1069` | `routers/sync.py:462` | `utils/sync/playback.py`, S3 `BUCKET_PRIVATE_CLOUD_SYNC` | in |
| `GET /v1/sync/audio/{id}/urls` | `audio.dart:151`, called from `pages/conversation_detail/widgets/conversation_bottom_bar.dart:508` and `services/audio_download_service.dart:36` | `routers/sync.py:487` | Presigned URLs on `BUCKET_PRIVATE_CLOUD_SYNC` | in |
| `GET /v1/conversations/{id}/recording-speakers` | loader at `conversation_detail_provider.dart:75`, reached from the transcript tab | `routers/recording_speakers.py:39` | `u/conversations/{id}` transcript and `u/people` | in. 404 is handled as disabled |
| `POST /v1/conversations/{id}/calendar-event/auto-link` | `conversations.dart:371`, via `conversation_detail_provider.dart:1415` | `routers/conversations.py:1303` | Google Calendar integration rows | cut. Returns null on any non-200, and the calendar integration is disabled |
| `GET /v1/conversations/{id}/screenshots` | fetched from `widgets/summary_tab.dart:193` | `routers/screen_frames.py:341` | `screen_frames` subcollection (`database/screen_frames.py:39`), `BUCKET_SCREEN_FRAMES` (unset) | cut. On `ApiFailure` the section hides itself (`conversation_screenshots_section.dart:132`) |
| `GET /v2/apps/search?installed_apps=true` | `backend/http/api/apps.dart:187` `retrieveInstalledChatApps`, via `fetchInstalledSummaryApps` (`conversation_summary_generation.dart:74`) and `fetchAndCacheEnabledConversationApps` (228), called at `conversation_detail_provider.dart:1065` | `routers/apps.py:696` | App catalog and the user's enabled apps | cut. Router `apps` disabled. Failure sets `SummaryTemplatesLoad.failed` and keeps the cache |
| `POST /v1/conversations/search` | `conversations.dart:1323`, called from `pages/search/global_search.dart:163` | `routers/conversations.py:2399`, then `utils/conversations/search.py:342` | Typesense | cut. Route disabled by `OMI_DISABLED_ROUTES`, so 404. The client returns `ConversationSearchResult.failure` (`conversations.dart:1312`) |
| `GET /v1/conversations/hidden-noise-counts` | `conversations.dart:462`. No caller in `lib/` outside the API file | `routers/conversations.py:1069` | `u/conversations` aggregation | not reached |

## 5. Memories

`MEMORY_ENABLED=on` is required. Memory fails closed without it
(`config/memory_rollout.py`). The list cursor is signed with
`MEMORY_V3_CURSOR_SECRET` (`utils/memory/universal_list_cursor.py`). Memory
writes go through `utils/memory/memory_service.py::MemoryService`, which also
calls `utils/memory/canonical_vector_sync.py`. That sync targets Pinecone, which
is cut and becomes a no-op.

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `GET /v3/memories?…cursor` | `backend/http/api/memories.dart:225` `getMemoriesResult` (URL from `buildMemoriesListUrl`, 138), called from `providers/memories_provider_loading.dart:10` | `routers/memories.py:711` | `u/memory_items` (cursor-paged), `u/memory_review_queue` | in |
| `POST /v3/memories` | `memories.dart:22` `createMemoryServer`, called from `providers/memories_provider_deletion.dart:141` and `pages/onboarding/guided_voice_io.dart:27` | `routers/memories.py:478`, then `MemoryService.create_external_memory` | `u/memory_items`, `u/memory_operations` (transaction). LLM `memory_category` | in |
| `PATCH /v3/memories/{id}` | `memories.dart:459` `editMemoryServer`, called from `providers/memories_provider.dart:136` | `routers/memories.py:1159` | `u/memory_items/{id}`, `u/memory_operations` | in |
| `PATCH /v3/memories/{id}/visibility?value=…` | `memories.dart:47` `updateMemoryVisibilityServer`, called from `memories_provider.dart:134` | `routers/memories.py:1196` | `u/memory_items/{id}` (`visibility`), `u/memory_operations` | in |
| `DELETE /v3/memories/{id}` | `memories.dart:382` `deleteMemoryServer`, called from `memories_provider_deletion.dart:83` | `routers/memories.py:1073` | `destructive_operation_gate`, `u/memory_items/{id}`, `u/memory_operations` | in |
| `DELETE /v3/memories` | `memories.dart:395` `deleteAllMemoriesServer`, called from `memories_provider.dart:132` and `pages/memories/widgets/memory_management_sheet.dart:254` | `routers/memories.py:1091` | `destructive_operation_gate`, all of `u/memory_items`, `u/memory_operations` | in |
| `GET /v3/memories/ledger-history` | `memories.dart:337`, fetched on every list load | `routers/memories.py:892` | `u/memory_operations` | in. Non-200 gives `supported: false` |
| `POST /v3/memories/{id}/review` | `memories.dart:502` | `routers/memories.py:1114` | `u/memory_items/{id}`, `u/memory_review_queue`, `u/memory_operations` | in |
| `POST /v3/memories/{id}/revert` | `memories.dart:424` | `routers/memories.py:1135` | `u/memory_items/{id}`, `u/memory_operations` | in |
| `PATCH /v3/memories/{id}/baseline` | `memories.dart:485`, called from `memories_provider.dart:989` | `routers/memories.py:1251` | `u/memory_items/{id}`, `u/memory_operations` | in |

## 6. Chat

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `POST /v2/messages?app_id&chat_session_id` (streaming) | `backend/http/api/messages.dart:258` `sendMessageStreamServer`, called from `providers/message_provider.dart:995` | `routers/chat.py:250`, then `utils/chat_turn.py::run_chat_turn` (829), then `utils/retrieval/graph.py::execute_chat_stream` (227) | See the chain below | in |
| `GET /v2/messages?chat_session_id&offset&limit` | `backend/http/api/chat_sessions.dart:87` `messages`, called from `providers/chat_history_state.dart:80,130,254`. Legacy loader `messages.dart:80` `getMessagesServer`, called from `chat_history_state.dart:270` | `routers/chat.py:371` | `u/messages`, filtered by `chat_session_id` | in |
| `DELETE /v2/messages?app_id&chat_session_id` | `messages.dart:115` `clearChatServer`, called from `chat_history_state.dart:312` | `routers/chat.py:322` | `u/messages` (batch delete), `u/chat_sessions/{id}` | in |
| `POST /v2/initial-message?app_id` | `messages.dart:322` `getInitialAppMessage`, called from `message_provider.dart:1191` and `pages/chat/page.dart:888` | `routers/chat.py:361`, then `utils/chat.py::initial_message_util` (70) | `u/chat_sessions`, `u/messages`, `users/{uid}` (language). LLM `chat_responses` (`utils/llm/chat.py`) | in |
| `POST /v2/chat-sessions` | `chat_sessions.dart:70` `create`, called from `chat_history_state.dart:153` | `routers/chat_sessions.py:130` | `u/chat_sessions` | in |
| `DELETE /v2/chat-sessions/{id}` | `chat_sessions.dart:80` `delete`, called from `chat_history_state.dart:110` | `routers/chat_sessions.py:172` | `u/chat_sessions/{id}`, `u/messages` with that session id | in |
| `POST /v2/chat/generate-title` | `chat_sessions.dart:91` `title`, called from `chat_history_state.dart:172` | `routers/chat_sessions.py:345` | LLM via `get_llm`, writes `u/chat_sessions/{id}` (`title`), `u/llm_usage` | in |
| `POST /v2/files` | `messages.dart:373`, via `providers/message_provider.dart:575` `uploadFiles` from the chat attachment control | `routers/chat.py:1518` | See the upload chain below | in |
| `POST /v2/files/video` | `messages.dart:396`, same caller for video attachments | `routers/chat_video.py:57` | See the upload chain below | in |
| `POST /v1/chat/receipts/{id}/undo` | `pages/chat/widgets/edit_receipt_block.dart:100` | `routers/chat_receipts.py:13`, then `utils/messaging/undo.py:20` `UndoStore` | `u/chat_receipts` (`undo.py:37`) and the record the receipt names | in |
| `POST /v2/tts/synthesize` | `backend/http/api/tts.dart`, via the Omi voice playback service | `routers/tts.py:93` | Hosted TTS provider | cut. Router `tts` disabled. Non-200 returns null, `TtsUnavailableException` is raised, and playback falls back to the system voice (`services/voice_playback/omi_voice_playback_service.dart`) |
| `POST /v2/voice-messages`, `POST /v2/voice-message/transcribe`, `WS /v2/voice-message/transcribe-stream` | `providers/voice_recorder_provider.dart:322,382` | `routers/chat.py:421`, `:562`, `:929` | Batch and streaming STT | cut. Routes disabled by `OMI_DISABLED_ROUTES`. The recorder catches the failure |
| `GET /v2/chat-sessions` | `chat_sessions.dart:66` `list`, called only from the channel chats page | `routers/chat_sessions.py:138` | `u/chat_sessions` | not reached (messaging channels cut) |

**`POST /v2/files` and `/v2/files/video` upload chain:**

1. **Images and documents.** `utils/other/chat_file.py:249` `FileChatTool.upload`
   sends the file to OpenAI Files (`openai.files.create`, line 292). The
   returned id is stored as `openai_file_id`.
2. **Thumbnails.** `utils/other/storage.py:2343` `upload_multi_chat_files`
   writes them to `BUCKET_CHAT_FILES` and returns public URLs.
3. **Rows.** `database/chat.py:701` `add_multi_files` writes `u/files/{id}`.
4. **Video.** `utils/media/video_upload.py:54` `_transcribe_file` calls
   `transcribe_pcm_bytes` (`utils/chat.py:257`). A prerecorded STT failure
   becomes `MediaPreparationError`, which `utils/media/media.py:258` catches.
   The transcript is empty and the upload still succeeds. The filmstrip is also
   sent to OpenAI Files (`chat_file.py:343`).
5. **Answer time.** `_ask_files_stream` (`chat_file.py:397`) and `_ask_files`
   (462) call OpenAI chat completions directly and read `files.content`.

After the port the OpenAI Files steps move to S3 plus Bedrock content blocks
(ADR-001, "Chat files"). Video transcripts stay empty in the MVP.

**`POST /v2/messages` dependency chain:**

1. **Auth and rate limit.** Redis.
2. **Quota.** `utils/subscription.py` reads `users/{uid}.subscription`. The
   owner is seeded with `plan=unlimited`. The chat quota does not honour
   `MARKETPLACE_APP_REVIEWERS`.
3. **Turn setup.** `run_chat_turn` writes the user message to `u/messages`
   (`chat_db.add_message`), links `u/chat_sessions/{id}`
   (`chat_db.add_message_to_chat_session`), reads attachments from `u/files`
   and S3 `BUCKET_CHAT_FILES`, and reads history
   (`chat_db.get_cache_aligned_messages`).
4. **Agent.** `utils/retrieval/agentic.py::_run_openai_agent_stream` calls
   `get_llm('chat_agent')`. After the port that is Bedrock Converse
   `au.anthropic.claude-sonnet-4-5-20250929-v1:0`, streamed through
   `tool_call_chunks`.
5. **Tools.** The conversation tools read `u/conversations`. Their vector
   branch, `vector_db.query_vectors` in
   `utils/retrieval/tool_services/conversations.py:330`, is skipped while
   Pinecone is cut. The memory tools read `u/memory_items`. The calendar, Gmail,
   Perplexity and app tools report unconfigured.
6. **Output.** Streams the reply, writes the AI message to `u/messages`, and
   records `u/llm_usage`. Goal extraction takes a Redis lock
   (`try_acquire_goal_extraction_lock`) and runs LLM `goals`.

## Checked, not reached

These routes were followed from an MVP screen and found unreachable on the MVP
path, so they need no row above.

| Route | Why it is not reached |
| --- | --- |
| `GET /v1/fair-use/status` (`users.dart:1007`, `routers/fair_use_admin.py:271`) | Called from `sync_upload_gate.dart:127` and `main.dart:638` only when persisted fair-use state exists. `FAIR_USE_ENABLED=false` never creates it |
| `GET /v1/users/me/usage` (`users.dart:611`) | Called only from the usage page |
| `GET /v1/sync/audio/{id}/{audio_file_id}` (`routers/sync.py:519`) | `getAudioStreamUrl` has no caller outside the API file |
| `GET /v1/conversations/{id}/suggested-apps` (`routers/conversations.py:2555`) | Reached only from the app summary sheet, which needs the cut app store |
| `POST /v2/messages/{id}/report` (`routers/chat.py:308`) | No caller outside the API file |
| `GET /v1/conversations/hidden-noise-counts` | No caller outside the API file |

## Cut and deferred features

Cut routes are disabled by a named switch. ADR-001's
[cut switches](ADR-001-aws-service-mapping.md#cut-switches) section defines
each switch, who enforces it, and the response a client gets.

**Cut (disabled by a switch):**

| Area | Routes | Switch |
| --- | --- | --- |
| Firebase sign-in | `/v1/auth/*` | `OMI_DISABLED_ROUTERS=auth` |
| Push | `POST /v1/users/fcm-token` | `OMI_DISABLED_ROUTES` entry |
| Offline and SD-card sync upload | `/v2/sync-local-files*`, `/v2/sync-capture-manifest` | `OMI_DISABLED_ROUTES` entries. The WAL keeps audio on 404 |
| Search | `POST /v1/conversations/search` | `OMI_DISABLED_ROUTES` entry |
| Apps and plugins | `/v1/apps*`, `/v2/apps*`, app reviews, developer routes | `OMI_DISABLED_ROUTERS=apps,developer,developer_key` |
| Billing writes | `/v1/payments/*`, `/v1/payment-methods/*`, retention, yearly switch, redeem code, storefront | `OMI_DISABLED_ROUTERS=payment,retention_offer,yearly_switch,redeem_code,storefront`. `GET /v1/users/me/subscription` stays in |
| Calendar and X | `/v1/calendar/*`, calendar onboarding, X connector | `OMI_DISABLED_ROUTERS=google_calendar,calendar_onboarding,x_connector` |
| Integration OAuth starts | `GET /v1/task-integrations/{app_key}/oauth-url`, `GET /v1/integrations/{app_key}/oauth-url` | `OMI_DISABLED_ROUTES` entries. The status reads stay mounted |
| Phone calls | `/v1/phone/*` | `OMI_DISABLED_ROUTERS=phone_calls`. `/v4/listen?source=phone_call` cannot start without `/v1/phone/token` |
| Messaging channels | `/v1/messaging/*` | `OMI_DISABLED_ROUTERS=messaging` |
| TTS | `/v2/tts/*` | `OMI_DISABLED_ROUTERS=tts` |
| Voice messages | `/v2/voice-messages`, `/v2/voice-message/transcribe`, `WS /v2/voice-message/transcribe-stream` | `OMI_DISABLED_ROUTES` entries |
| Speech-profile enrolment | `/v4/speech-profile`, `/v3/speech-profile/expand`, `stt-availability` | `OMI_DISABLED_ROUTES` entries. `GET /v3/speech-profile` stays and returns `false` |

**Deferred but served.** Goals, folders, the knowledge graph, wrapped and
referrals read the same Firestore adapter. They stay mounted and work once the
adapter lands, but they are not MVP acceptance criteria.
