---
name: finalcad-export-datasets-to-bi
description: Configure and download Finalcad One's daily Parquet datasets for Power BI or any warehouse — the supported bulk-read path, and the right alternative to paging the transactional API.
api: Finalcad One API
base_url: https://developer.finalcad.cloud/api
generated: '2026-08-17'
method: generated
source: openapi/finalcad-organizations-openapi.yml, https://developer.finalcad.com/
operations:
  - userOrganizationsGetOrganizationId
  - datasetsGetDatasets
  - datasetsPostDatasets
  - datasetsGetDatasetContent
  - datasetsGetDatasetUrl
---

# Export Finalcad One datasets for BI

Finalcad generates a flattened analytics view of the whole platform once a day and hands it over as
Parquet. For reporting, dashboards or a warehouse load, this is the supported path — one request per
dataset instead of thousands of paged API calls.

## Steps

1. **Find your organization.** `userOrganizationsGetOrganizationId` — `GET /organizations`.

2. **See what is enabled.** `datasetsGetDatasets` —
   `GET /organizations/{organization_id}/data/config`. Returns every dataset with an `is_active`
   flag.

3. **Turn on what you need.** `datasetsPostDatasets` —
   `POST /organizations/{organization_id}/data/config` with
   `{"data_sets": ["Observations", "Forms", "Modules", "Phases", "Statuses", "Projects"]}`.
   The response echoes the full list with the new `is_active` values.

   Datasets Finalcad documents: `Observations`, `Statuses`, `CommonObservations`, `FormInstances`,
   `FormInstanceStatuses`, `FormTemplates`, `Plans`, `Companies`, `Phases`, `Modules`, `Users`,
   `Projects`, `Workspaces`, plus `Priorities` and `UserActivities` (added API 2.35) and
   `ProjectsUsersRoles` (added API 2.38, users with their role in each project).

4. **Get a download URL.** `datasetsGetDatasetUrl` —
   `GET /organizations/{organization_id}/data/config/dataseturl?name=Projects`. Returns a
   time-limited URL; fetch the Parquet from it. `datasetsGetDatasetContent` (`.../dataset?name=`)
   is the sibling that returns content directly.

5. **Wire it into Power BI.** Finalcad publishes a Power Query snippet and a dashboard template.
   The pattern is a `Web.Contents` call to `dataseturl` carrying `X-API-Key` and
   `Authorization: token <key>` headers, wrapped in a retry, then `Parquet.Document` over the
   returned URL. The connection to the returned URL itself is anonymous — Power BI will prompt about
   that, and the prompt is expected, because the authentication happened on the metadata call.

## Timing

Datasets regenerate **once a day at 06:00**. A dataset pull is therefore up to 24 hours stale by
design. If you need same-day numbers, use the differential feed instead
(see `finalcad-sync-observations`) and accept the request volume.

## Personal data

`Users`, `UserActivities` and `ProjectsUsersRoles` contain named individuals and their activity.
Finalcad is a French company operating under GDPR and its data is covered by the Orisha group
privacy policy. Treat these three datasets as personal data: restrict who can query them, do not
copy them into ad-hoc spreadsheets, and set a retention period on whatever you load them into.
The other datasets are project data and are not subject to the same handling.

## Failure handling

- A dataset URL is time-limited. Do not persist it — persist the dataset name and re-request.
- If a file has not been generated yet (a newly enabled dataset before the next 06:00 run), the URL
  call can fail; Finalcad's own Power BI example wraps it in a three-attempt retry with a two-second
  delay for exactly this reason.
- Errors follow the standard envelope `{"statut", "api_code", "message", "data"}`.
