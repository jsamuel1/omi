# MVP API surface for the AWS port

Companion to [ADR-001](ADR-001-aws-service-mapping.md). This census places
every backend route the Flutter app (`app/`) can call into exactly one bucket:
**in** (the MVP port must serve it), **cut** (a switch in ADR-001 removes it and
the client tolerates the 404), or **not reached** (no MVP screen or startup path
calls it). The MVP loop is sign-in and bootstrap, device and firmware, live
capture, the Home tab, conversations, memories and chat. Rows in sections 1 to 6
give the client caller, the backend handler, what the handler touches, and the
bucket.

Line references are to `origin/main` at `a04153d180`. Client paths are relative
to `app/lib/`, and backend paths to `backend/`. A route is "in" when the MVP
port must serve it correctly. It is "cut" when its feature is disabled by a
switch in ADR-001's [cut switches](ADR-001-aws-service-mapping.md#cut-switches)
and the client keeps working without it.

## How the census was built

The census is a total partition, not a sample. It was built mechanically, then
each route was classified by reading its callers.

1. **Routes.** Every `@router` decorator under `backend/routers/`: 777
   decorated routes. Each is keyed by method, path
   and owning module (`Mount.module` in `app_roles.py`).
2. **Client calls.** Every backend URL the app builds: each `Env.apiBaseUrl` or
   `Env.authApiBaseUrl` string under `app/lib/` (excluding `l10n/` and generated
   files), with its HTTP method, plus the socket clients under
   `services/sockets/` and `services/connectivity_service.dart`. Helper-built
   URLs (`_baseUrl`, `conversationCollectionUrl`, `buildMemoriesListUrl`,
   `account_cutover_control_client.dart`) were followed by hand. The first pass
   gave 291 unique method+path pairs.
3. **Second pass.** A grep of every `'v[1-4]/…'` and `'memory/…'` literal under
   `app/lib/`, plus the helper-built URLs, added 24 pairs the first pass missed
   (among them the static map, health probe, knowledge graph, memory search,
   calendar events, referral campaign, payment webviews, developer and MCP
   keys, and the speech-profile upload). The census therefore covers 315
   client method+path pairs. Each was matched to its decorator. Five strings
   match no decorator: one real call and four URL builders or debug strings,
   all listed under [No backend route](#no-backend-route).
4. **Chains.** For each matched handler, the `database.*` modules it imports to
   depth 3 give the collections it touches. The shared auth baseline (Redis,
   `users`, `client_devices`) is subtracted. Rows below name the stores the
   handler itself writes or reads.
5. **Reach.** For every pair, the client function's callers were followed to a
   mounted page or a startup path. A route is **in** when it is called on a
   signed-in startup, by a mounted MVP screen, or by a primary control on one.
   It is **cut** when its feature is cut and the client was checked to tolerate
   a 404. Every other pair is **not reached** and is listed, with its reason,
   in the [partition appendix](#partition-appendix).

**MVP screens.** Startup (`main.dart`, `core/app_shell.dart:420`
`_initializeProviders`, `mobile/mobile_app.dart`, first-run onboarding), the
Home tab (the conversation list, episodes, the Activity page opened from the
bell at `pages/home/page.dart:1085`, the global search overlay, the merge bar
and folder actions), conversation detail, capture, memories, chat, and device
settings. The Tasks tab, Goals, the settings pages other than device settings,
daily recaps, the conversations-with-tasks page and the conversation map are
outside the MVP.

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

**Startup order.** Before any product traffic, `main.dart:383-387` binds the
signed-in owner with `AccountCutoverRuntime.bindAuthenticatedOwner`, which
fetches `GET /v1/account/cutover/control`. `AccountCutoverBlockingGate`
(`mobile/mobile_app.dart:67`, `pages/home/page.dart:102`) holds the product UI
until that control is read, so this route fails closed. `_initializeProviders`
(`core/app_shell.dart:420`) then awaits `GET /v1/users/deletion-status`. It returns `false`, and startup stops,
unless that call is an `ApiSuccess` with no pending deletion (line 429). It then
starts, without awaiting, the speech-profile check, the language list, the
user and people providers, the message refresh, the subscription fetch, and the
task-integration and integration loads (lines 431-451).

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `GET /v1/account/cutover/control` | `services/account_cutover/account_cutover_control_client.dart:62-63` `_defaultFetch`, via `AccountCutoverRuntime.bindAuthenticatedOwner` (`account_cutover_runtime.dart:129`), called at `main.dart:383-387` and on resume | `routers/account_cutover.py:19` `get_account_cutover_control` | `users/{uid}/account_cutover/state` (`database/account_cutover.py`), with a Redis epoch key. A missing document means legacy-compatible. A malformed one fails closed | in. Startup blocks on it |
| `GET /v1/health` | `services/connectivity_service.dart:193`, the connectivity probe on non-production profiles | `routers/other.py:8` | None | in. The fork's build profile is non-production, so this replaces the public probes |
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
| `GET /v1/episodes/live` | `backend/http/api/episodes.dart:30`, via `EpisodeProvider.refreshLive` (516) | `routers/episodes.py:85` | `u/episodes`, `u/conversations`. Returns `{enabled: false}` unless the client channel is admitted (see the note below) | in |
| `GET /v1/episodes?from&to&limit&cursor` | `episodes.dart:68-76` `getEpisodes`, via `EpisodeProvider.refreshHome` (136, page fetch at 200), called from `pages/conversations/conversations_page.dart:414,754,886` | `routers/episodes.py:61` `list_episodes`, then `utils/episodes/home_read.py::list_episode_window` | `u/episodes`, `u/episode_links`, `u/conversations` (`database/episode_timeline.py`), Redis client-seen marker (`database/episodes_client_seen.py`). `schedule_missed_composes` (`home_read.py:389`) may run the composer, LLM `episode_composer` | in |
| `POST /v1/conversations/task-presence` | `backend/http/api/conversation_tasks.dart:31-33`, via `ConversationTaskPresenceProvider` (20), requested for each visible row at `pages/conversations/widgets/conversation_list_item.dart:379-380` | `routers/conversation_tasks.py:25` `task_presence`, then `database/conversation_task_links.py::visible_task_presence` | `u/action_items`, `u/conversations`. Catches `GoogleAPICallError` and answers 503 | in |
| `POST /v1/announcements/{announcement_id}/dismiss` | `backend/http/api/announcements.dart:86`, via `AnnouncementProvider.markAnnouncementDismissed` (47), called at `services/announcement_service.dart:158` after an explicit answer | `routers/announcements.py:177` | `u/dismissed_announcements` (`database/announcements.py:487`) | in |
| `POST /v1/telemetry/home-rows` | `pages/conversations/home_rows_health.dart:45` `_send`, from `HomeRowsHealth.note` at `conversations_page.dart:866` | `routers/client_telemetry.py:125`, then `utils/home_rows_metric.py:40` | In-process counter only | in |
| `GET /v1/folders` | `backend/http/api/folders.dart:10`, via `FolderProvider.loadFolders` (35), called from the folder actions at `pages/conversations/conversation_actions.dart:148,232` and `pages/conversation_detail/widgets/conversation_detail_header.dart:289` | `routers/folders.py:31` | `u/folders`, `u/port_mutations` (`database/ports/folders.py`) | in |
| `POST /v1/folders`, `PATCH /v1/folders/{folder_id}`, `DELETE /v1/folders/{folder_id}` | `folders.dart:29,65,81`, via `FolderProvider` (70, 96, 129), from `pages/conversations/widgets/create_folder_sheet.dart:107,120` and `folder_options_sheet.dart:64` | `routers/folders.py:57`, `:85`, `:108` | `u/folders`, `u/port_mutations`, and the moved `u/conversations` on delete | in |
| `PATCH /v1/conversations/{conversation_id}/folder`, `POST /v1/folders/{folder_id}/conversations/bulk-move` | `folders.dart:95,108`, via `FolderProvider` (152, 170), from the conversation actions and the merge bar | `routers/folders.py:174`, `:201` | `u/conversations` (`folder_id`), `u/folders` counts | in |
| `POST /v1/conversations/merge` | `conversations.dart:1492` `mergeConversations`, via `ConversationProvider` (2095), from the Home merge bar | `routers/conversations.py:2664`, then `utils/conversations/merge_conversations_job.py` in a background task | `conversation_finalization_jobs` (`database/conversation_merge_jobs.py:19`), `u/conversations`, `u/deleted_conversations`. With `reprocess`, the section 3 step 9 LLM chain | in |
| `POST /v1/attention-items/seen` | `pending_work.dart:85`, via `PendingWorkStore.markSeen` (234), called when the Activity page shows (`pages/activity/activity_page.dart:183`) | `routers/pending_work.py:185` | `u/processing_attention_items` (`database/processing_attention.py`) | in |
| `POST /v1/attention-items/{item_id}/retry`, `POST /v1/attention-items/retry-all` | `pending_work.dart:61,72`, via `PendingWorkStore.retry` (161) and `retryAll` (202), from `pages/activity/needs_attention_section.dart:136,109` | `routers/pending_work.py:195`, `:126` | `u/processing_attention_items`, `u/processing_retry_claims`, then `POST /v1/conversations/{conversation_id}/reprocess`'s chain | in |
| `GET /v1/phone/numbers` | `backend/http/api/phone_calls.dart:55`, via `PhoneCallProvider.loadVerifiedNumbers` (177). The provider is lazy (`main.dart:767`) and is first read by `pages/home/widgets/battery_info_widget.dart:306` when the record control is tapped; its constructor calls `loadVerifiedNumbers` (154) | `routers/phone_calls.py:208` | `u/phone_numbers` | cut. `OMI_DISABLED_ROUTERS=phone_calls`. The provider catches the failure and keeps an empty list (`phone_call_provider.dart:184-187`) |
| `GET /v1/review/open-loops`, `GET /v2/review/items`, `PUT /v1/users/release-channel` | `backend/http/api/open_loops.dart:24`, `review.dart:22`, `review.dart:108`, via `ReviewProvider._load` (152) for `ReviewEntryCard` at `conversations_page.dart:920`, and `_reportChannel` (129) | `routers/review.py:114`, `:74`, `routers/dream_cohort.py:21` | Review stores (`review_*`) | cut. `OMI_DISABLED_ROUTERS=review,dream_cohort`. A 404 or 403 sets `ReviewAvailability.off` (`review_provider.dart:172-176`), the card hides, and a release-channel failure is only logged (130) |
| `POST /v1/goals/canonical` | `backend/http/api/goals.dart:293` `createIntroductionGoal`, from the voice-profile flow's goal step (`pages/onboarding/guided_voice_io.dart:209-212`, `guided_voice_controller.dart:673`). The flow opens from the Home speech-profile card (`pages/capture/widgets/widgets.dart`, `conversations_page.dart:917`) through `openVoiceProfile` (`settings_destinations.dart:160`). With iOS on-device speech the flow reaches this step even though the profile upload is cut | `routers/goals.py:88` | `u/workstreams`, `u/action_items`, `u/memory_state` (`database/goals.py`) | in |
| `GET /v1/review/changes` | `review.dart:42`, loaded by the Activity page (`activity_page.dart:178`) | `routers/review.py:162` | `u/review_changes` | cut. `OMI_DISABLED_ROUTERS=review`. The page treats 404 as "no changes" (`activity_page.dart:192-193`) |
| `GET /v1/announcements/pending` | `backend/http/api/announcements.dart:50`, via `services/announcement_service.dart:61`, triggered at `home/page.dart:603` | `routers/announcements.py:119` | `announcements`, `u/dismissed_announcements` (`database/announcements.py:479`) | in |
| `GET /v1/announcements/changelogs` | `announcements.dart:12`, via `announcement_service.dart:81` | `routers/announcements.py:35` | `announcements` | in |

**Episodes admission.** `routers/episodes.py` answers `{enabled: false}` unless
`EPISODES_ENABLED` is on (the default) and the request's app channel is listed
in `EPISODES_CLIENT_CHANNELS` (default `beta`, `config/episodes.py:18-25`). The
app sends its channel from `utils/app_channel.dart:31`. The fork sets
`EPISODES_CLIENT_CHANNELS=beta,store` so both its debug and release builds get
the episode UI. With episodes off, the detail, wrap-up and dismiss routes below
are never called, so the decision does not change what must be served.

### Episodes

The episode page (`pages/episodes/episode_page.dart`), the live episode page
(`pages/episodes/live_episode_page.dart`) and the episode section of
conversation detail (`summary_tab.dart:203` `EpisodeMomentsSection`) call these
through `EpisodeProvider` (`providers/episode_provider.dart:32-37`).

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `GET /v1/episodes/{episode_id}` | `episodes.dart:43`, via `EpisodeProvider.openEpisode` (599, detail at 635), from `episode_page.dart:239` | `routers/episodes.py:124` | `u/episodes/{id}`, `u/conversations` | in |
| `GET /v1/episodes/by-conversation/{conversation_id}` | `episodes.dart:36`, via `EpisodeProvider.episodeFor` (573), from `conversation_list_item.dart:255` and `pages/episodes/episode_moments_section.dart:34,43` | `routers/episodes.py:105` | `u/episode_links`, `u/episodes`, Redis client-seen marker | in |
| `POST /v1/episodes/{episode_id}/wrap-up` | `episodes.dart:51`, via `EpisodeProvider.wrapUp` (683) and `onLiveFinalized` (719), called from `live_episode_page.dart:130` and `services/capture/capture_controller.dart:4158` | `routers/episodes.py:142` | `u/episodes/{id}` (closes it, which queues the composer) | in |
| `POST /v1/episodes/{episode_id}/items/{item_id}/dismiss` | `episodes.dart:62`, via `EpisodeProvider.dismissItem` (728), from `live_episode_page.dart:301` | `routers/episodes.py:156` | `u/episodes/{id}` items | in |
| `PATCH /v1/episodes/{episode_id}/assign-speaker` | `backend/http/api/recording_speakers.dart:40`, the episode branch of `assignRecordingSpeakers`, from `coordinators/provider_capture_external_actions.dart:120` | `routers/episodes.py:170` | `u/episodes/{id}`, `u/conversations` transcript segments | in |

### Search overlay

The global search overlay (`pages/search/global_search.dart`) is mounted from
Home. Its direct calls are in. The pages it routes to (conversations with tasks,
daily recaps, the conversation map) are outside the MVP.

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `GET /v1/search/overview` | `backend/http/api/search.dart:66`, from `global_search.dart:158` | `routers/search.py:63` | Counts over `u/conversations`, `u/folders`, `u/memory_items`, daily summaries, `u/people` | in |
| `GET /v1/users/daily-summaries/search` | `search.dart:76`, from `global_search.dart:176` | `routers/users.py:2168` | Daily summaries (`database/daily_summaries.py`) | in |
| `GET /v1/action-items/search` | `search.dart:93`, from `global_search.dart:210` | `routers/action_items.py:708` | `u/action_items`, then a Pinecone query, which is cut and returns empty | in |
| `GET /memory/search` | `search.dart:109-111`, from `global_search.dart:213` and `pages/memories/page.dart:106` | `routers/memory_product.py:108` | `u/memory_items` and the memory ledger modules | in |

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
| `PATCH /v1/conversations/{conversation_id}/segments/assign-bulk` | `backend/http/api/conversations.dart:753` `assignBulkConversationTranscriptSegments`, called from `pages/conversation_detail/widgets/transcript_tab.dart:300` and `conversation_detail_provider.dart:70` | `routers/conversations.py:1990`, then `_assign_manual_speaker` (1858) | `u/conversations/{id}` (transcript segments, in a transaction), `u/people` | in |

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
| `GET /v1/conversations/{conversation_id}` | `conversations.dart:588` `getConversationById`, called from `pages/conversation_detail/conversation_detail_provider.dart:87` and `widgets/summary_tab.dart:71` | `routers/conversations.py:1136` | `u/conversations/{id}` | in |
| `POST /v1/conversations` (process the in-progress conversation now) | `conversations.dart:48` `processInProgressConversationOutcome`, called from `services/capture/capture_controller.dart:4186` | `routers/conversations.py:503` | `lifecycle_service.admit_processing`, then the finalization chain of section 3 steps 8 and 9. Redis cached geolocation | in |
| `POST /v1/conversations/{conversation_id}/reprocess` | `conversations.dart:235` `reProcessConversationServer`, called from `conversation_detail_provider.dart:86` and `pages/activity/needs_attention_section.dart:277` | `routers/conversations.py:759` | `u/conversations/{id}`, then `process_conversation` (LLM features above, `u/memory_items`). Hourly limit in Redis | in |
| `PATCH /v1/conversations/{conversation_id}/title?title=…` | `conversations.dart:641` `updateConversationTitle`, called from `conversation_detail_provider.dart:1008` | `routers/conversations.py:1194` | `u/conversations/{id}` (`title`) | in |
| `PATCH /v1/conversations/{conversation_id}/starred?starred=…` | `conversations.dart:780` `setConversationStarred`, called from `pages/conversations/conversation_actions.dart:177` and `pages/conversation_detail/page.dart:837` | `routers/conversations.py:2218` | `u/conversations/{id}` (`starred`), `u/people`, Redis conversation-owner key | in |
| `DELETE /v1/conversations/{conversation_id}?cascade=true` | `conversations.dart:325` `deleteConversationServer`, called from `providers/conversation_provider.dart:1643` | `routers/conversations.py:1436` | `destructive_operation_gate` (legal-hold and deletion gate docs), `u/conversations/{id}` and its `photos` subcollection (`database/conversations.py:2594`), memory retraction in `u/memory_items` (`memory_service.retract_conversation_memories`), S3 audio prefixes (`utils/other/audio_packs.py:126`), Pinecone delete (cut, so a no-op) | in |
| `POST /v1/sync/audio/{conversation_id}/precache` | `backend/http/api/audio.dart:136-139`, called from `conversation_detail_provider.dart:1069` | `routers/sync.py:462` | `utils/sync/playback.py`, S3 `BUCKET_PRIVATE_CLOUD_SYNC` | in |
| `GET /v1/sync/audio/{conversation_id}/urls` | `audio.dart:151`, called from `widgets/conversation_bottom_bar.dart:508` and `services/audio_download_service.dart:36` | `routers/sync.py:487` | Presigned URLs on `BUCKET_PRIVATE_CLOUD_SYNC` | in |
| `GET /v1/conversations/{conversation_id}/recording-speakers` | loader at `conversation_detail_provider.dart:75`, reached from the transcript tab | `routers/recording_speakers.py:39` | `u/conversations/{id}` transcript and `u/people` | in. 404 is handled as disabled |
| `PATCH /v1/conversations/{conversation_id}/segments/text` | `conversations.dart:657`, via `conversation_detail_provider.dart:831` | `routers/conversations.py:1400` | `u/conversations/{id}` segments, `u/transcript_corrections`, `u/review_changes` (`database/review_transcript_changes.py`) | in |
| `PATCH /v1/conversations/{conversation_id}/summary` | `conversations.dart:668`, via `conversation_detail_provider.dart:906` | `routers/conversations.py:1381` | `u/conversations/{id}` structured summary. The Typesense index write is skipped while `TYPESENSE_HOST` is unset | in |
| `PATCH /v1/conversations/{conversation_id}/visibility` | `conversations.dart:768`, from `widgets/conversation_detail_header.dart:338` and `widgets/share_to_contacts_sheet.dart:177` | `routers/conversations.py:2014` | `u/conversations/{id}` (`visibility`) | in |
| `POST /v1/conversations/{conversation_id}/capture-group/separate` | `conversations.dart:797`, via `pages/conversation_detail/capture_group_separation.dart:13` | `routers/conversations.py:1366` | `u/conversations/{id}` capture-group fields | in |
| `PATCH /v1/conversations/{conversation_id}/action-items` | `conversations.dart:813`, from `widgets/conversation_action_items_section.dart:361` | `routers/conversations.py:1695` | `u/conversations/{id}` action items, `u/action_items` | in |
| `PATCH /v1/conversations/{conversation_id}/action-items/{action_item_idx}` | `conversations.dart:831`, from `pages/action_items/widgets/action_item_form_sheet.dart:112`, opened from detail at `conversation_action_items_section.dart:413` | `routers/conversations.py:1783` | `u/conversations/{id}`, `u/action_items`, `u/workstreams` (`database/ports/action_items.py`) | in |
| `DELETE /v1/conversations/{conversation_id}/action-items` | `conversations.dart:843`, from `conversation_detail_provider.dart:1038` and `providers/conversation_provider.dart:1862` | `routers/conversations.py:1820`. The same path is also decorated at `routers/action_items.py:1013`, but `conversations` mounts first (`app_roles.py:196` before 202), so this handler serves it | `u/conversations/{id}`, `u/action_items` | in |
| `POST /v1/action-items`, `GET /v1/action-items/{action_item_id}`, `PATCH /v1/action-items/{action_item_id}`, `DELETE /v1/action-items/{action_item_id}` | `backend/http/api/action_items.dart:275,206,340,361`, from `action_item_form_sheet.dart:125,262` and `conversation_action_items_section.dart:351`, `ActionItemsProvider.updateActionItemState` (611) | `routers/action_items.py:376`, `:760`, `:771`, `:907` | `u/action_items`, `u/workstreams`, `u/port_mutations` (`database/ports/action_items.py`). The vector write is cut | in |
| `POST /v1/action-items/share` | `action_items.dart:396`, from `action_item_form_sheet.dart:186` | `routers/action_items.py:1112` | `u/action_items`, Redis share token (`redis_db.store_task_share`) | in |
| `PATCH /v1/conversations/{conversation_id}/recording-speakers` | `recording_speakers.dart:39`, from `coordinators/provider_capture_external_actions.dart:120` and `conversation_detail_provider.dart` | `routers/recording_speakers.py:61` | `u/conversations/{id}` transcript segments, `u/people` | in |
| `PATCH /v1/conversations/{conversation_id}/assign-speaker/{speaker_id}` | `recording_speakers.dart:97` `_assignLegacy`, the fallback when `recording-speakers` answers 404, and `conversations.dart:754` | `routers/conversations.py:1955` | `u/conversations/{id}` | in |
| `POST /v1/conversations/{conversation_id}/speakers/{speaker_id}/reject` | `backend/http/api/speaker_labels.dart:43`, from `widgets/transcript_tab.dart:306` and `pages/episodes/live_transcript_view.dart` | `routers/speaker_labels.py:43` | `u/conversations/{id}`, `u/people`, `u/speaker_learning_jobs`, recurring-voice stores | in |
| `POST /v1/users/people` | `users.dart:347`, from `conversation_detail_provider.dart:388` and `providers/people_provider.dart:146` (tagging a new speaker) | `routers/users.py:663` | `u/people`, `u/port_mutations` (`database/ports/people.py`) | in |
| `GET /v1/users/people/{person_id}`, `GET /v1/users/people/{person_id}/voice-matches` | `speaker_labels.dart:57,70`, from `widgets/speaker_tag_outcome.dart:53-54` | `routers/users.py:685`, `routers/speaker_labels.py:120` | `u/people`, `u/speaker_tag_prompts` (`database/voice_profiles.py`), signed URLs on `BUCKET_SPEECH_PROFILES` | in |
| `GET /v1/speaker-tag-prompts/clip` | `backend/http/api/speaker_tag_prompts.dart:73`, from `widgets/earlier_voice_matches_sheet.dart:185` | `routers/speaker_tag_prompts.py:92` | `u/conversations/{id}`, audio from `BUCKET_PRIVATE_CLOUD_SYNC` via the audio timeline | in |
| `DELETE /v1/users/preferences/app` | `users.dart:591`, via `writeServerPreferredSummaryApp` (`conversation_summary_generation.dart:71`) when the reader picks Omi's summary | `routers/users.py:1134` | `users/{uid}` (preferred app) | in |
| `POST /v1/mobile/feedback` | `users.dart:144`, from `widgets/summary_tab.dart:240,256` `SummaryFeedbackPrompt` | `routers/mobile_feedback.py:53` | `feedback_reports`, `feedback_events` (`database/feedback.py`) | in |
| `POST /v1/conversations/{conversation_id}/calendar-event/auto-link` | `conversations.dart:373`, via `conversation_detail_provider.dart:1415` | `routers/conversations.py:1302` | `u/integrations` (Google Calendar), LLM | cut. `OMI_DISABLED_ROUTES` entry, because the route lives in the retained `conversations` module. The provider returns null on any failure (1424) |
| `POST /v1/conversations/{conversation_id}/calendar-event`, `DELETE /v1/conversations/{conversation_id}/calendar-event` | `conversations.dart:354,341`, from `widgets/calendar_event_sheets.dart:123` and the event chip (`conversation_detail_header.dart:192`) | `routers/conversations.py:1241`, `:1203` | `u/integrations`, `u/conversations/{id}` | cut. `OMI_DISABLED_ROUTES` entries. "Link event" opens the sheet only when Google Calendar is connected (`pages/conversation_detail/page.dart:698-706`), which it never is |
| `GET /v1/calendar/google/events` | `conversations.dart:402` `listGoogleCalendarEvents`, from `conversation_detail_provider.dart:1451` | `routers/google_calendar.py:112` | Google Calendar API | cut. `OMI_DISABLED_ROUTERS=google_calendar`. The provider returns an empty list on failure |
| `GET /v1/conversations/{conversation_id}/screenshots` | fetched from `widgets/summary_tab.dart:193` | `routers/screen_frames.py:341` | `screen_frames` subcollection (`database/screen_frames.py:39`), `BUCKET_SCREEN_FRAMES` | cut. `OMI_DISABLED_ROUTERS=screen_frames`. On `ApiFailure` the section hides itself (`conversation_screenshots_section.dart:132-135`) |
| `GET /v1/conversations/{conversation_id}/photos/{photo_id}/image` | `conversations.dart:630`, via `widgets/conversation_photo_image.dart:25` | `routers/frame_requests.py:66`, then `services/conversation_frame_evidence.py:15` | `BUCKET_FRAME_REQUESTS` (`utils/retrieval/frame_request_storage.py:25`) | cut. `OMI_DISABLED_ROUTERS=frame_requests`. The necklace has no camera, and the image cache returns null on failure (`conversation_photo_image.dart:41-54`) |
| `GET /v1/conversations/{conversation_id}/entities` | `review.dart:87`, via `ReviewProvider.conversationEntities` (136), from `widgets/conversation_entity_chips.dart` (`conversation_detail_header.dart:101`) | `routers/review.py:193` | Review stores | cut. `OMI_DISABLED_ROUTERS=review`. With Review off the provider returns an empty list before calling (137) |
| `GET /v1/static-map` | `widgets/omi_map_preview.dart:61`, from `pages/conversation_detail/widgets.dart:574` | `routers/static_map.py:31`, then `utils/static_map.py:343` | Google Maps Static API through `GOOGLE_MAPS_API_KEY` | cut. `OMI_DISABLED_ROUTERS=static_map`. The image's `errorWidget` renders the pin-dot canvas (`omi_map_preview.dart:167-175`) |
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
| `PATCH /v3/memories/{memory_id}` | `memories.dart:459` `editMemoryServer`, called from `providers/memories_provider.dart:136` | `routers/memories.py:1159` | `u/memory_items/{id}`, `u/memory_operations` | in |
| `PATCH /v3/memories/{memory_id}/visibility?value=…` | `memories.dart:47` `updateMemoryVisibilityServer`, called from `memories_provider.dart:134` | `routers/memories.py:1196` | `u/memory_items/{id}` (`visibility`), `u/memory_operations` | in |
| `DELETE /v3/memories/{memory_id}` | `memories.dart:382` `deleteMemoryServer`, called from `memories_provider_deletion.dart:83` | `routers/memories.py:1073` | `destructive_operation_gate`, `u/memory_items/{id}`, `u/memory_operations` | in |
| `DELETE /v3/memories` | `memories.dart:395` `deleteAllMemoriesServer`, called from `memories_provider.dart:132` and `pages/memories/widgets/memory_management_sheet.dart:254` | `routers/memories.py:1091` | `destructive_operation_gate`, all of `u/memory_items`, `u/memory_operations` | in |
| `GET /v3/memories/ledger-history` | `memories.dart:337`, fetched on every list load | `routers/memories.py:892` | `u/memory_operations` | in. Non-200 gives `supported: false` |
| `POST /v3/memories/{memory_id}/review` | `memories.dart:502` | `routers/memories.py:1114` | `u/memory_items/{id}`, `u/memory_review_queue`, `u/memory_operations` | in |
| `POST /v3/memories/{memory_id}/revert` | `memories.dart:424` | `routers/memories.py:1135` | `u/memory_items/{id}`, `u/memory_operations` | in |
| `PATCH /v3/memories/{memory_id}/baseline` | `memories.dart:485`, called from `memories_provider.dart:989` | `routers/memories.py:1251` | `u/memory_items/{id}`, `u/memory_operations` | in |
| `GET /v1/knowledge-graph`, `GET /v1/knowledge-graph/canonical` | `backend/http/api/knowledge_graph_api.dart:41-42` `getKnowledgeGraph`, loaded by `MemoriesPage` (`pages/memories/page.dart:28`, `showMindMap` defaults to true) and `MemoryMindMapPreview` (334) | `routers/knowledge_graph.py:172`, `:217`, then `utils/memory/canonical_graph.py` | `u/memory_state` (`database/knowledge_graph.py`), `u/memory_items` | in |

## 6. Chat

| Method and path | Client caller | Backend handler | Store / providers | MVP |
| --- | --- | --- | --- | --- |
| `POST /v2/messages?app_id&chat_session_id` (streaming) | `backend/http/api/messages.dart:258` `sendMessageStreamServer`, called from `providers/message_provider.dart:995` | `routers/chat.py:250`, then `utils/chat_turn.py::run_chat_turn` (829), then `utils/retrieval/graph.py::execute_chat_stream` (227) | See the chain below | in |
| `GET /v2/messages?chat_session_id&offset&limit` | `backend/http/api/chat_sessions.dart:87` `messages`, called from `providers/chat_history_state.dart:80,130,254`. Legacy loader `messages.dart:80` `getMessagesServer`, called from `chat_history_state.dart:270` | `routers/chat.py:371` | `u/messages`, filtered by `chat_session_id` | in |
| `DELETE /v2/messages?app_id&chat_session_id` | `messages.dart:115` `clearChatServer`, called from `chat_history_state.dart:312` | `routers/chat.py:322` | `u/messages` (batch delete), `u/chat_sessions/{id}` | in |
| `POST /v2/initial-message?app_id` | `messages.dart:322` `getInitialAppMessage`, called from `message_provider.dart:1191` and `pages/chat/page.dart:888` | `routers/chat.py:361`, then `utils/chat.py::initial_message_util` (70) | `u/chat_sessions`, `u/messages`, `users/{uid}` (language). LLM `chat_responses` (`utils/llm/chat.py`) | in |
| `POST /v2/chat-sessions` | `chat_sessions.dart:70` `create`, called from `chat_history_state.dart:153` | `routers/chat_sessions.py:130` | `u/chat_sessions` | in |
| `DELETE /v2/chat-sessions/{session_id}` | `chat_sessions.dart:80` `delete`, called from `chat_history_state.dart:110` | `routers/chat_sessions.py:172` | `u/chat_sessions/{id}`, `u/messages` with that session id | in |
| `POST /v2/chat/generate-title` | `chat_sessions.dart:91` `title`, called from `chat_history_state.dart:172` | `routers/chat_sessions.py:345` | LLM via `get_llm`, writes `u/chat_sessions/{id}` (`title`), `u/llm_usage` | in |
| `POST /v2/files` | `messages.dart:373`, via `providers/message_provider.dart:575` `uploadFiles` from the chat attachment control | `routers/chat.py:1518` | See the upload chain below | in |
| `POST /v2/files/video` | `messages.dart:396`, same caller for video attachments | `routers/chat_video.py:57` | See the upload chain below | in |
| `POST /v2/messages/device-tool/{call_id}/result` | `services/device_tools/device_tool_dispatcher.dart:152` `_postResult`, created per streamed reply at `messages.dart:283` | `routers/chat.py:273`, then `utils/device_tools.py:183` | Redis in-flight and result keys with a TTL | in |
| `POST /v1/users/analytics/chat_message` | `users.dart:485` `setMessageResponseRating`, via `MessageProvider.setMessageNps` (609), from the rating control at `pages/chat/page.dart:1408` | `routers/users.py:840` | `u/messages/{id}` (`rating`) | in |
| `POST /v1/chat/receipts/{receipt_id}/undo` | `pages/chat/widgets/content_blocks/edit_receipt_block.dart:100` | `routers/chat_receipts.py:13`, then `utils/messaging/undo.py:20` `UndoStore` | `u/chat_receipts` (`undo.py:37`) and the record the receipt names | in |
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
   `ChatMediaStore.transcribe_pcm`, which calls `transcribe_pcm_bytes`
   (`utils/chat.py:257`). A prerecorded STT failure becomes
   `MediaPreparationError` (`utils/messaging/adapters/media.py:265`), which
   `_transcribe_file` catches (`video_upload.py:60`).
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

## Cut routes and their switches

Every route the census marks cut is named here with the exact switch entry that
removes it. Router switches name a `Mount.module` from `app_roles.py`. Route
switches name a decorator's method and path exactly. ADR-001's
[cut switches](ADR-001-aws-service-mapping.md#cut-switches) section defines each
switch, who enforces it, and the response a client gets.

| Area | Routes the app can call | Switch entry |
| --- | --- | --- |
| Firebase sign-in | `/v1/auth/*` (`POST /v1/auth/token`, `POST /v1/auth/local-dev/custom-token`, `/v1/auth/authorize`) | `OMI_DISABLED_ROUTERS=auth` |
| Push | `POST /v1/users/fcm-token` | `OMI_DISABLED_ROUTES` `POST /v1/users/fcm-token` |
| Offline and SD-card sync upload | `POST /v2/sync-capture-manifest`, `POST /v1/sync-local-files`, `POST /v2/sync-local-files`, `POST /v2/sync-local-files/{job_id}/release-hold`, `GET /v2/sync-local-files/{job_id}` | `OMI_DISABLED_ROUTES`, one entry each. The WAL keeps audio on 404 |
| Search | `POST /v1/conversations/search` | `OMI_DISABLED_ROUTES` `POST /v1/conversations/search` |
| Apps and plugins | `/v1/apps*`, `/v2/apps*`, `/v1/app/*`, `/v1/app-*`, app reviews and keys, `/v1/dev/*` | `OMI_DISABLED_ROUTERS=apps,developer,developer_key` |
| Billing | `/v1/payments/*`, `/v1/stripe/*`, `/v1/payment-methods/*`, `/v1/paypal/*`, `/v1/apps/{app_id}/subscription`, retention offer, yearly switch, redeem code, `/v2/payments/plans` | `OMI_DISABLED_ROUTERS=payment,retention_offer,yearly_switch,redeem_code,storefront`. `GET /v1/users/me/subscription` stays in |
| Google Calendar and X | `/v1/calendar/google/*`, `/v1/calendar/capture-gaps`, `/v1/calendar/onboarding/*`, `/v1/calendar/meetings*`, `/v1/x/*` | `OMI_DISABLED_ROUTERS=google_calendar,calendar_onboarding,calendar_meetings,x_connector` |
| Calendar links on conversations | `POST /v1/conversations/{conversation_id}/calendar-event`, `DELETE /v1/conversations/{conversation_id}/calendar-event`, `POST /v1/conversations/{conversation_id}/calendar-event/auto-link` | `OMI_DISABLED_ROUTES`, one entry each. These live in the retained `conversations` module |
| Integration OAuth starts | `GET /v1/task-integrations/{app_key}/oauth-url`, `GET /v1/integrations/{app_key}/oauth-url` | `OMI_DISABLED_ROUTES`, one entry each. The status reads stay mounted |
| Phone calls | `/v1/phone/*`, including `GET /v1/phone/numbers` | `OMI_DISABLED_ROUTERS=phone_calls`. `/v4/listen?source=phone_call` cannot start without `POST /v1/phone/token` |
| Messaging channels | `/v1/messaging/*` | `OMI_DISABLED_ROUTERS=messaging` |
| TTS | `/v2/tts/*`, `/v1/tts/voices`, `/v1/users/voice` | `OMI_DISABLED_ROUTERS=tts` |
| Voice messages | `POST /v2/voice-messages`, `POST /v2/voice-message/transcribe`, `WS /v2/voice-message/transcribe-stream` | `OMI_DISABLED_ROUTES`, one entry each |
| Speech-profile enrolment | `GET /v4/speech-profile`, `POST /v3/upload-audio`, `GET /v3/speech-profile/expand`, `DELETE /v3/speech-profile/expand`, `GET /v3/speech-profile/stt-availability` | `OMI_DISABLED_ROUTES`, one entry each. `GET /v3/speech-profile` stays and returns `false` |
| Review and dream | `/v1/review/*`, `/v2/review/*`, `/v1/entities*`, `GET /v1/conversations/{conversation_id}/entities`, `PUT /v1/users/release-channel`, `/v1/dream/runs` | `OMI_DISABLED_ROUTERS=review,dream_cohort,dream_report`. Already off by `REVIEW_SURFACE_MODE` and `DREAM_AGENT_ENABLED=false`; the switch makes the 404 deterministic |
| Static map | `GET /v1/static-map` | `OMI_DISABLED_ROUTERS=static_map` |
| Frame and photo evidence | `GET /v1/conversations/{conversation_id}/photos/{photo_id}/image`, `/v1/frame-requests*` | `OMI_DISABLED_ROUTERS=frame_requests` |
| Screenshots | `GET /v1/conversations/{conversation_id}/screenshots` and the other `screen_frames` routes | `OMI_DISABLED_ROUTERS=screen_frames` |

**Verification.** ADR-001's cut-switch verification test boots the app with the
fork's switch values and asserts, for every route in this table, that the route
is absent from `app.router.routes`. It also asserts that every route listed
**in** in sections 1 to 6 is still mounted. The table and the test share one
list, so a route cannot be marked cut here and survive in the app.

## No backend route

These client calls match no decorator. They fail today, in upstream, the same
way they fail in the port.

| Client call | Caller | Effect |
| --- | --- | --- |
| `POST /v1/onboarding/seed-conversation` | `conversations.dart:1516`, from `pages/onboarding/variants/pocket/pocket_onboarding_flow.dart:100` | Returns `false` on any non-2xx (`conversations.dart:1521-1523`). Onboarding continues; Home opens without a seeded conversation |
| `GET /v1/dev` | `backend/http/api/dev_api.dart:8` is a base URL, not a call. Its real calls are `/v1/dev/keys`, below | None |
| `GET /v1/users/{path}` | `account_deletion.dart:82` `_url` is a URL builder for `deletion-status` and `delete-account` | None |
| `PATCH /v1/apps/${appData[…]}` | A debug string in `apps.dart` | None |
| `GET /v1/payments/yearly-switch?preview` | `payment.dart:109`; the route is `GET /v1/payments/yearly-switch` (`yearly_switch.py:31`) with a query parameter | Cut with `yearly_switch` |

## Partition appendix

Every client method+path the census found that is not a row in sections 1 to 6.
Each is **cut** (its switch is in the table above) or **not reached** (with the
reason). Grouped by client file.

| Client file | Routes | Bucket and reason |
| --- | --- | --- |
| `backend/http/api/users.dart` | `GET /v1/fair-use/status` (`users.dart:1007`, `routers/fair_use_admin.py:271`) | not reached. Called from `sync_upload_gate.dart:127` and `main.dart:638` only when persisted fair-use state exists, and `FAIR_USE_ENABLED=false` never creates it |
| `backend/http/api/users.dart` | `GET /v1/users/me/usage` (`users.dart:611`) | not reached. Usage page only |
| `backend/http/api/conversations.dart` | `GET /v1/conversations/{conversation_id}/suggested-apps` (`routers/conversations.py:2554`) | not reached. Only the app summary sheet calls it, and that needs the cut app catalog |
| `backend/http/api/messages.dart` | `POST /v2/messages/{message_id}/report` (`routers/chat.py:308`) | not reached. No caller outside the API file |
| `backend/http/api/review.dart`, `open_loops.dart` | `POST /v2/review/items/{item_id}/answer`, `POST /v1/review/open-loops/{date}/{card_id}/answer` | cut. `review`. With Review off the cards that answer never render |
| `backend/http/plan_storefront_api.dart` | `GET /v2/payments/plans` | cut. `storefront` |
| `backend/http/api/action_items.dart` | `GET /v1/action-items/pending-sync`, `PATCH /v1/action-items/sync-batch` | not reached. Apple Reminders sync (`services/integrations/apple_reminders_sync_service.dart`) is a Tasks-tab integration |
| `backend/http/api/action_items.dart` | `GET /v1/action-items/shared/{token}`, `POST /v1/action-items/accept` | not reached. Only the shared-tasks deep link (`core/app_shell.dart:264`) and its accept sheet call them |
| `backend/http/api/action_items.dart` | `PATCH /v1/action-items/batch`, `POST /v1/action-items/batch-delete` | not reached. Task reorder (`ActionItemsProvider._flushSortUpdates`) and bulk delete (`task_selection_action_bar.dart:164`) are Tasks-tab controls |
| `backend/http/api/apps.dart`, `backend/http/api/payment.dart` (app subscriptions) | every `/v1/apps*`, `/v2/apps*`, `/v1/app/*`, `/v1/app-*` route, including `POST /v1/apps/enable` from `conversation_summary_generation.dart:327`, `POST /v1/app/thumbnails` and `POST /v1/apps/migrate-owner` (Google or Apple account linking, `auth_provider.dart:385,407`) | cut. `apps`, `developer`, `developer_key`, `payment` |
| `backend/http/api/assistant_voices.dart` | `GET /v1/tts/voices`, `GET /v1/users/voice`, `PATCH /v1/users/voice` | cut. `tts`. Only the voice settings page calls them |
| `backend/http/api/calendar_capture_telemetry.dart` | `GET /v1/calendar/capture-gaps` | cut. `google_calendar`. The capture gap monitor only asks when `google_calendar_connected` is set (`services/capture/capture_composition.dart:149-155`), and treats failure as no evidence (157) |
| `backend/http/api/conversation_tasks.dart` | `POST /v1/conversations/with-tasks/page` | not reached. Only the conversations-with-tasks page calls it, opened from search at `global_search.dart:692` |
| `backend/http/api/conversations.dart` | `GET /v1/conversations/{conversation_id}/transcripts`, `GET /v1/conversations/hidden-noise-counts` | not reached. No caller outside the API file |
| `backend/http/api/conversations.dart` | `POST /v1/conversations/{conversation_id}/test-prompt` | not reached. Shown only in the developer tools group (`pages/conversation_detail/page.dart:934,1059`) |
| `backend/http/api/conversations.dart` | `POST /v2/sync-capture-manifest`, `GET /v2/sync-local-files/{job_id}` | cut. `OMI_DISABLED_ROUTES` entries |
| `backend/http/api/audio.dart` | `GET /v1/sync/audio/{conversation_id}/{audio_file_id}` | not reached. `getAudioStreamUrl` has no caller outside the API file |
| `backend/http/api/daily_summary_creation.dart`, `users.dart` (daily summaries), `pages/conversations/widgets/daily_summaries_list.dart` | `POST /v1/users/daily-summaries`, `GET /v1/users/daily-summaries/{summary_id}`, `PATCH /v1/users/daily-summaries/{summary_id}/visibility`, `DELETE /v1/users/daily-summaries/{summary_id}`, `POST /v1/users/daily-summaries/{summary_id}/regenerate`, `GET`/`PATCH /v1/users/daily-summary-settings` | not reached. Home's recap row is a developer opt-in (`conversations_page.dart:921-929`), and the recap detail and settings pages are outside the MVP. The Activity page lists recaps but a fresh account has none |
| `backend/http/api/dev_api.dart`, `backend/http/api/mcp_api.dart` | `/v1/dev/keys*`, `/v1/mcp/keys*`, `GET /v1/mcp`, `GET`/`DELETE /v1/mcp/oauth/grants*` | not reached. Developer settings only |
| `backend/http/api/dream.dart` | `GET /v1/dream/runs`, `POST /v1/dream/runs` | cut. `dream_report` |
| `backend/http/api/folders.dart` | `POST /v1/folders/reorder` | not reached. `FolderProvider.reorderFolders` (185) has no UI caller |
| `backend/http/api/goals.dart` | every other `/v1/goals*` route (the canonical create is in section 1) | not reached. Goals live on the Goals page and the Tasks tab |
| `backend/http/api/imports.dart` | `GET /v1/import/jobs`, `DELETE /v1/import/limitless/conversations` | not reached. Import history settings page only |
| `backend/http/api/integrations.dart` | `PUT /v1/integrations/{app_key}`, `DELETE /v1/integrations/{app_key}`, `PUT /v1/integrations/apple-health/sync` | not reached. Integrations settings and the Apple Health service only |
| `backend/http/api/knowledge_graph_api.dart` | `POST /v1/knowledge-graph/rebuild` | not reached. Only `prebuildForOnboarding` (133) calls it, from the control onboarding arm (`pages/onboarding/wrapper.dart:177`); release builds use the Pocket arm (`onboarding_variant_gate.dart:109`) |
| `backend/http/api/messaging_channels.dart` | every `/v1/messaging/*` route | cut. `messaging` |
| `backend/http/api/payment.dart`, `payments.dart`, `budget_choice_api.dart`, `plan_storefront_api.dart`, `pages/settings/payment_webview_page.dart` | every `/v1/payments/*`, `/v1/payment-methods/*`, `/v1/stripe/*`, `/v1/paypal/*` and `/v2/payments/plans` route | cut. `payment`, `retention_offer`, `yearly_switch`, `redeem_code`, `storefront`. The chat budget pause reads the storefront only when two PostHog flags are on (`services/experiments/plan_storefront_gate.dart:13-16`), which fail to `false` without PostHog |
| `backend/http/budget_choice_api.dart` | `GET`/`PUT /v1/users/budget-choice` | not reached. The freemium switch dialog and the non-default onboarding arms only |
| `backend/http/api/pending_work.dart` | `POST /v1/attention-items/{item_id}/dismiss` | not reached. No caller outside the API file |
| `backend/http/api/phone_calls.dart` | `POST /v1/phone/numbers/verify`, `POST /v1/phone/numbers/verify/check`, `DELETE /v1/phone/numbers/{phone_number_id}`, `POST /v1/phone/token` | cut. `phone_calls` |
| `backend/http/api/privacy.dart` | `/v1/users/migration/*` | not reached. `UserProvider.updateDataProtectionLevel` (468) and `getMigrationCountFor` (458) have no UI caller |
| `backend/http/api/referrals.dart` | `GET /v1/users/me/referral`, `GET /v2/users/me/referral/campaign`, `POST /v2/users/me/referral/campaign/claim` | not reached. The referral page (`pages/referral/referral_page.dart`) is opened only from settings |
| `backend/http/api/review.dart` | `GET /v1/entities`, `GET /v1/entities/{entity_id}/page`, `POST /v1/entities/{entity_id}/corrections`, `POST /v1/review/changes/{change_id}/undo`, `POST /v1/review/changes/{change_id}/redo` | cut. `review` |
| `backend/http/api/speaker_tag_prompts.dart` | `GET /v1/speaker-tag-prompts`, `POST /v1/speaker-tag-prompts/shown`, `/dismiss`, `/answer`, `/ignored-voices*`, `GET`/`PATCH /v1/users/voice-profile-settings` | not reached. `SpeakerTagPromptsProvider.loadIfDue` (175) has no external caller, and the settings are read only by `voice_profile_settings_section.dart:38` |
| `backend/http/api/speech_profile.dart`, `pages/onboarding/guided_voice_requests.dart`, `guided_voice_io.dart` | `GET /v4/speech-profile`, `POST /v3/upload-audio`, `GET`/`DELETE /v3/speech-profile/expand`, `GET /v3/speech-profile/stt-availability`, plus `POST /v2/voice-message/transcribe` from the guided voice flow | cut. `OMI_DISABLED_ROUTES` entries. The voice-profile flow (`openVoiceProfile`, `settings_destinations.dart:160`) is opened from the Home speech-profile card. Its preflight treats a 404 as unavailable and shows `transcription_unavailable` (`guided_voice_controller.dart:258-264,313-315`); the reader can skip |
| `backend/http/api/task_integrations.dart` | `PUT /v1/task-integrations/{app_key}`, `PUT /v1/task-integrations/default`, `DELETE /v1/task-integrations/{app_key}`, `POST /v1/task-integrations/{app_key}/tasks`, and the Asana and ClickUp listing routes | not reached. `TaskIntegrationsPage` is opened from the Tasks tab header (`pages/home/page.dart:993-1007`) and settings |
| `backend/http/api/task_suggestions.dart` | every `/v1/candidates*` route | not reached. `TaskSuggestionsController.load` is called from the Tasks tab (`action_items_page.dart:102`). `POST /v1/candidates/from-conversation` runs only for desktop-source conversations (`summary_task_recovery.dart`) |
| `backend/http/api/users.dart` | `PATCH /v1/users/people/{person_id}/name`, `PATCH /v1/users/people/{person_id}/pinned`, `DELETE /v1/users/people/{person_id}`, `DELETE /v1/users/people/{person_id}/speech-samples/{sample_index}` | not reached. People settings pages only (`pages/settings/people.dart`, `person_detail_page.dart`, `widgets/people_list.dart`) |
| `backend/http/api/users.dart` | `POST /v1/users/private-cloud-sync`, `POST /v1/users/training-data-opt-in` | not reached. The private cloud sync page and the training-data settings only |
| `backend/http/api/users.dart` | `POST /v1/users/processing-backlog/process` | not reached. `UsageProvider.processBacklog` has no UI caller |
| `backend/http/api/users.dart` | `GET`/`POST`/`DELETE /v1/users/store-recording-permission`, `GET`/`POST /v1/users/analytics/memory_summary` | not reached. No caller outside the API file |
| `backend/http/api/users.dart` | `/v1/users/developer/webhook*`, `GET`/`PATCH /v1/users/mentor-notification-settings` | not reached. Developer and notification settings only |
| `backend/http/api/users.dart` | `DELETE /v1/users/delete-account` | not reached. Profile settings only |
| `backend/http/api/users.dart` | `PUT /v1/users/preferences/app` | not reached. It sets a summary template, and the template list comes from the cut app catalog (`fetchInstalledSummaryApps` fails), so no template can be chosen |
| `backend/http/api/wrapped.dart` | `GET /v1/wrapped/{year}`, `POST /v1/wrapped/{year}/generate` | not reached. The Wrapped page opens only from a deep link (`home_deep_links.dart:279`) |
| `backend/http/user_data_export.dart` | `GET /v1/users/export` | not reached. Data export settings only |
| `pages/settings/device_diagnostics.dart` | `POST /v1/mobile/device-diagnostics` | not reached. "Send to support" on the diagnostics page under device settings; a manual support action. If used, it writes to `BUCKET_PRIVATE_CLOUD_SYNC` (`utils/other/device_diagnostics_storage.py:14`) and works unchanged |
| `services/on_device/on_device_segment_upload.dart` | `POST /v1/conversations/from-segments` | cut. `developer`. On-device transcription defaults off, so `on_device_runner.dart` never uploads |
| `backend/http/api/phone_calls.dart:109` | the `source=phone_call` variant of `/v4/listen` | cut with `phone_calls`. The phone call page needs `POST /v1/phone/token` first, so it cannot open the socket |

**Deferred but served.** Goals, the knowledge graph rebuild, wrapped, referrals
and the settings pages above are not reached on the MVP path. Their routers stay
mounted and read the same Firestore adapter, so they work once the adapter
lands, but they are not MVP acceptance criteria.
