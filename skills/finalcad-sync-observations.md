---
name: finalcad-sync-observations
description: Keep an external system in step with Finalcad One site observations using the continuous_token differential feed — the correct way to poll Finalcad for what changed, rather than re-listing everything.
api: Finalcad One API
base_url: https://developer.finalcad.cloud/api
generated: '2026-08-17'
method: generated
source: openapi/finalcad-projects-openapi.yml, openapi/finalcad-medias-openapi.yml, conventions/finalcad-conventions.yml
operations:
  - observationsGetProjectObservationsList
  - observationsGetFilteredObservationIds
  - observationsGetObservationDetails
  - observationsGetObservationMessages
  - projectLibrariesGetProjectObservationStatus
  - projectLibrariesGetProjectObservationPriorities
  - projectLibrariesGetProjectTrades
  - projectLibrariesGetProjectCommonObservations
  - getMediaResourceUrl
---

# Sync Finalcad One observations into an external system

Observations are Finalcad's punch-list items — defects, tasks and remarks raised on site. This is
the read-side integration: pull them into a defect tracker, a data warehouse, or a client report.

## The one thing to understand first

Finalcad's pagination and its change feed are **the same mechanism**. Every differential endpoint
returns:

```json
{ "need_to_relaunch": true, "continuous_token": "0|10", "count": 10, "total_count": 25 }
```

- While `need_to_relaunch` is `true`, call again with the same `limit` and the returned
  `continuous_token`. That is paging.
- Once `need_to_relaunch` is `false`, **store the last `continuous_token`**. Calling again later
  with that token returns only the observations added, modified or deleted since — that is the
  change feed. There is no separate "since" endpoint and no separate webhook needed for this.

Token format observed in the published examples is `"<ticks>|<offset>"`. Treat it as opaque.

## Steps

1. **Cache the project libraries once per run.** Observations reference ids, not labels, so resolve
   them or your downstream records are meaningless UUIDs:
   - `projectLibrariesGetProjectObservationStatus` — `GET /projects/{project_id}/status`
   - `projectLibrariesGetProjectObservationPriorities` — `GET /projects/{project_id}/observations-priorities`
   - `projectLibrariesGetProjectTrades` — `GET /projects/{project_id}/trades`
   - `projectLibrariesGetProjectCommonObservations` — `GET /projects/{project_id}/commonobservations`

   Each library item carries a `names[]` array of `{language, translation}` — pick the translation
   for your reporting locale rather than assuming one name.

2. **First full pull.** `observationsGetProjectObservationsList` —
   `GET /projects/{project_id}/observations?limit=50`. Loop while `need_to_relaunch` is true,
   passing `continuous_token` each time. Persist the final token against the project.

3. **Every run after that.** Same call, same `limit`, with the **stored** token. An empty
   `observations[]` with `count: 0` means nothing changed — that is a normal, cheap answer.

4. **Fetch detail only for what changed.** `observationsGetObservationDetails` —
   `GET /projects/{project_id}/observations/{observation_id}`.

5. **Pull the thread if you need it.** `observationsGetObservationMessages` —
   `GET /projects/{project_id}/observations/{observation_id}/messages`, which accepts `message_id`
   and `get_newest` to walk a conversation incrementally.

6. **Resolve photos.** Observation media come back as media ids. `getMediaResourceUrl` —
   `GET /medias/{media_id}/url` — returns a downloadable URL. Fetch it promptly; treat the URL as
   short-lived and never store it as a permanent reference — store the `media_id`.

## Alternative: a targeted query instead of a feed

`observationsGetFilteredObservationIds` — `POST /projects/{project_id}/observations/filter?limit=` —
is a **read implemented as a POST**. It returns matching observation ids for a filter set. Use it
for a one-off report ("every open high-priority observation for this trade"); use the differential
feed for ongoing sync.

## Alternative: bulk instead of polling

If you are feeding a warehouse rather than a live tracker, the differential feed is the wrong tool.
Enable the `Observations` dataset on the organization (`datasetsPostDatasets`) and download the
daily Parquet file via `datasetsGetDatasetUrl`. It regenerates once a day at 06:00 — a day stale,
but one request instead of thousands.

## Field notes

- Timestamps come in two flavours. `created_at` / `updated_at` are when Finalcad's server saved the
  record; `client_created_at` / `client_updated_at` are when the field user actually acted. On a
  site with poor signal those can differ by hours. For "when did this defect happen", use the
  client timestamps.
- Observations may carry `position_x` / `position_y` (a pin on a plan) and `latitude` / `longitude`.
- `offset` paging is available but constrained: `offset` must be a multiple of `limit`, or the call
  returns 400 `INVALID_INSTANCE_STATE`.
- No rate limits are published and no `RateLimit-*` headers are returned. Be conservative: pace your
  loop and back off on any 5xx rather than assuming headroom.
