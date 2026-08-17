---
name: finalcad-upload-plans-and-documents
description: Push drawings, BIM models and documents from an EDM/CDE into Finalcad One — the chunked media upload flow and the two independent folder trees it feeds.
api: Finalcad One API
base_url: https://developer.finalcad.cloud/api
generated: '2026-08-17'
method: generated
source: openapi/finalcad-medias-openapi.yml, openapi/finalcad-projects-openapi.yml, conventions/finalcad-conventions.yml
operations:
  - uploadMedias
  - chunkUploadInit
  - chunkUploadUpload
  - chunkUploadTerminate
  - chunkUploadAbort
  - locationsCreateFolder
  - locationsCreatePlans
  - locationsUpdatePlan
  - locationsGetPlanTree
  - locationsCreatePlansFromDocument
  - documentsCreateDocumentOrFolders
  - documentsGetDocumentsAndFolders
---

# Upload plans and documents into Finalcad One

The "right blueprint at the right time" integration: keep an EDM / CDE and Finalcad One in step so
the drawing the field team opens is the drawing the design office released.

## Two trees, same word

Finalcad has **two independent folder hierarchies** and they share the name "folder":

- **Locations** — folders containing **plans** (`locationsCreateFolder`, `locationsCreatePlans`).
  This is what observations get pinned to.
- **Documents** — folders containing **documents** (`documentsCreateDocumentOrFolders`).

Finalcad states plainly that despite the similar names the two are completely independent. A folder
id from one tree is meaningless in the other. Decide which tree a file belongs in before you upload.

## Step 1 — upload the bytes

Media are uploaded once at application level and then referenced by `media_id`.

**Files at or under 5 MB** — one call. `uploadMedias` — `POST /medias/uploads`, multipart with the
binary in `stream`. Response: `{ "id", "file_name", "mime_type" }`.

**Files over 5 MB** — four steps:

1. `chunkUploadInit` — `POST /medias/uploadinit` with `{ "file_name", "total_bytes", "md5" }`
   (md5 of the **whole** file). Returns `chunk_id`, `media_id`, `min_chunk_size` (5242880) and
   `expires_after_secs` (3600).
2. `chunkUploadUpload` — `POST /medias/{media_id}/uploadappend`, once per part, multipart with
   `chunk_id`, `segment_index`, `last_chunk`, `stream` and the `md5` **of that part**.
   `segment_index` starts at **1**. Every part except the last must be exactly 5242880 bytes.
   Returns 204.
3. `chunkUploadTerminate` — `POST /medias/{media_id}/uploadterminate` with `{ "chunk_id" }`.
4. `chunkUploadAbort` — `POST /medias/{media_id}/uploadabort` — if you need to cancel.

The session expires an hour after init. For a large model on a poor connection, that is a real
constraint: size your parts and your retries against it, and abort explicitly rather than leaving
half-uploaded sessions behind.

Finalcad checks the file **signature**, not just the extension — a renamed file fails with
`FILE_EXTENSION`. Empty parts fail with `EMPTY_FILE`, a malformed multipart with `BAD_HEADER`, and a
part with no filename with `MISSING_FILE_NAME`.

## Step 2a — file it as a plan

1. `locationsCreateFolder` — `POST /projects/{project_id}/folders` with `{ "name", "parent_id" }`.
   Omit `parent_id` to create at the root. Depth is unlimited.
2. `locationsCreatePlans` — `POST /projects/{project_id}/plans` with
   `{ "name", "media_id", "folder_id" }`. Omit `folder_id` for the root.
3. `locationsUpdatePlan` — `PUT /projects/{project_id}/plans/{plan_id}` — for a **new revision** of
   an existing drawing. Update the plan rather than creating a second one, or the observations
   pinned to the original lose their context.

Accepted plan formats: PDF and DWG for drawings; IFC and RVT for 3D models via
`locationsCreatePlansFromDocument` and the BIM upload path.

## Step 2b — file it as a document

`documentsCreateDocumentOrFolders` — `POST /projects/{project_id}/documents`. The same endpoint
creates both a folder and a document depending on the body; Finalcad publishes an example of each.
Read back with `documentsGetDocumentsAndFolders` or, for one level,
`documentsGetDocumentsAndFoldersByParent` (`?parent_id=`).

Visibility rule worth knowing: project documents are visible to all project editors and admins;
guests and restricted guests see only documents shared into a group they belong to.

## Failure handling

- No idempotency key. If a create call times out after the upload succeeded, you will not know
  whether the plan was created — list the tree (`locationsGetPlanTree`) and match by name before
  retrying, or you will end up with duplicates that observations may already be pinned to.
- On an error, read the `api_code` suffix: `_ERR{n}` rolled back, `_WRN{n}` did not.
- A media that was uploaded but never attached is orphaned; there is no published cleanup endpoint.

## Do not use

The project-scoped media endpoints (`/projects/{project_id}/medias/...`) are marked **OBSOLETE** by
Finalcad — release note API 2.16, 22 January 2024 — in favour of the application-level ones above.
They are still published and still answer, with no sunset date, but new integrations should not use
them.
