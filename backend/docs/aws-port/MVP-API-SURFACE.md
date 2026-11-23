# MVP API surface for the AWS port

Companion to [ADR-001](ADR-001-aws-service-mapping.md). This census lists every
backend route the Flutter app (`app/`) calls for the MVP loop: sign-in and
bootstrap, device and firmware, live capture, conversations, memories and chat.
Each row gives the client caller, the backend handler, what the handler touches,
and whether the route is in the MVP.

Line references are to `origin/main` at `a04153d180`. Client paths are relative
to `app/lib/`, and backend paths to `backend/`. A route is "in" when the MVP
port must serve it correctly. It is "cut" when the client keeps working without
it.

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

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
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
| `POST /v1/users/fcm-token` | `notifications.dart:18` `saveFcmTokenServer`, called only from the FCM service | `routers/notifications.py:35` | `u/fcm_tokens` | cut (push cut) |

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
| `POST /v1/conversations/search` | `conversations.dart:1323`, called from `pages/search/global_search.dart:163` | `routers/conversations.py:2399`, then `utils/conversations/search.py:342` | Typesense | cut. Returns 500 because `search_conversations` raises without `TYPESENSE_HOST` |
| `GET /v1/conversations/hidden-noise-counts` | `conversations.dart:462`. No caller in `lib/` outside the API file | `routers/conversations.py:1069` | `u/conversations` aggregation | cut (unused) |

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
| `GET /v2/chat-sessions` | `chat_sessions.dart:66` `list`, called only from the channel chats page | `routers/chat_sessions.py:138` | `u/chat_sessions` | cut (messaging channels cut) |

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

## Out of the MVP

These routes have app callers but are not needed for the MVP loop. The app
tolerates their failure or never reaches them once the matching feature is
cut. Each switch is listed in ADR-001's cut list.

| Area | Routes | Reason |
| --- | --- | --- |
| Firebase sign-in | `/v1/auth/*` | Replaced by the Cognito hosted UI |
| Push | `POST /v1/users/fcm-token` | Push cut |
| Offline and SD-card sync upload | `/v2/sync-local-files`, `/v2/sync-capture-manifest`, `/v1/sync-jobs/*` | Phase 2. The WAL keeps audio on 503 (`services/wals/CUSTODY.md`) |
| Search | `POST /v1/conversations/search` | Typesense cut |
| Apps and plugins | `/v1/apps*`, `/v2/apps*`, app reviews | App store cut |
| Billing | `/v1/payments/*`, `/v1/payment-methods/*`, `GET /v1/users/me/subscription`, `GET /v1/users/me/usage` | Stripe cut. No MVP caller found for `getUserSubscription` or `getUserUsage` (`backend/http/api/users.dart:695,611`) |
| Integrations | `/v1/integrations/*`, task integrations, calendar | Third-party integrations cut |
| Phone calls | `/v1/phone/*`, `/v4/listen?source=phone_call` | Twilio cut |
| Messaging channels | `/v1/messaging/*`, `GET /v2/chat-sessions` | Telegram and iMessage adapters off |
| Speech-profile enrolment | `/v4/speech-profile`, `/v3/speech-profile/expand`, `stt-availability` | Owner voiceprint cut. `GET /v3/speech-profile` stays and returns `false` |
| Voice messages | `/v2/voice-messages`, `/v2/voice-message/transcribe` | Uses batch STT. Phase 2 |
| Action items, goals, folders, people, knowledge graph, daily summaries, wrapped, referrals | their `/v1/*` routes | Secondary pages. They read the same Firestore adapter and work once it lands, but they are not MVP acceptance criteria |
