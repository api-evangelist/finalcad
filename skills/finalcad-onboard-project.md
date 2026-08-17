---
name: finalcad-onboard-project
description: Create a Finalcad One construction project from an upstream system (ERP job, work order) and staff it with members, companies and phases — the standard "a project for each construction site" integration.
api: Finalcad One API
base_url: https://developer.finalcad.cloud/api
generated: '2026-08-17'
method: generated
source: openapi/finalcad-organizations-openapi.yml, openapi/finalcad-projects-openapi.yml, openapi/finalcad-libraries-openapi.yml, conventions/finalcad-conventions.yml
operations:
  - userOrganizationsGetOrganizationId
  - workspacesGetWorkspaces
  - languagesGetLanguages
  - getListOfTimeZones
  - projectDetailsCreateANewProject
  - projectMembersManagementGetProjectRoles
  - projectMembersManagementAddMembersToAProject
  - companiesCreateProjectCompanies
  - phasesCreatePhase
  - projectDetailsCheckProjectSettings
---

# Onboard a construction project into Finalcad One

Use this when an upstream system (ERP, project-control tool, job-costing system) opens a new job and
that job needs a Finalcad One project so field teams can start logging observations and forms.

## Before you start

- Both headers on every call: `X-API-Key: <organization key>` and `Authorization: token <api key>`.
- The organization must already exist and hold an Enterprise licence — the API cannot create one.
- Set `Accept-Language` if you want API responses in something other than English.

## Steps

1. **Find your organization.** `userOrganizationsGetOrganizationId` — `GET /organizations`.
   Returns `organizations[]` with `id` and `name`, plus `count` / `total_count` / `limit` / `offset`.
   Keep the `id`; nearly every later call is scoped by it.

2. **Pick a workspace (optional).** `workspacesGetWorkspaces` —
   `GET /organizations/{organization_id}/workspaces`. Accepts `parent_id` and `with_children`.
   A project created inside a workspace inherits that workspace's libraries instead of the
   organization's. Skip this step if the organization has no workspaces.

3. **Resolve language and time zone.** `languagesGetLanguages` — `GET /languages` — and
   `getListOfTimeZones` — `GET /time-zones?limit=&offset=`. Both are plain reference lists; the
   time-zone list is long (419 entries in the published example), so page it or filter client-side.
   Using a value that is not on these lists is rejected.

4. **Create the project.** `projectDetailsCreateANewProject` —
   `POST /organizations/{organization_id}/projects`.
   Only `name` is mandatory. Send `case_number` with **your** upstream job reference — that is the
   field Finalcad provides for the customer's own identification, and it is what makes the two
   systems reconcilable later. Also useful: `language`, `time_zone`, `description`, `address`,
   `start_date`, `end_date`, `workspace_id`.
   The response carries `project_id`, and the project has already inherited the organization's or
   workspace's trades, common observations, priorities and statuses.

5. **Read the project role list.** `projectMembersManagementGetProjectRoles` —
   `GET /projects/{project_id}/roles`. You need a `project_role_id` before you can add anyone.

6. **Add members.** `projectMembersManagementAddMembersToAProject` —
   `POST /projects/{project_id}/members`. Members may be identified by email or by id.
   Pass `doNotSendEmail=true` on the query string when you are bulk-provisioning from an HRIS or
   Active Directory and do not want every user to receive an invitation mail.
   This is a bulk write — see **Failure handling** below.

7. **Seed the company list.** `companiesCreateProjectCompanies` —
   `POST /projects/{project_id}/companies`. Push the subcontractors your purchasing system already
   knows about, each with its `client_reference` (your internal id). Field teams can then assign
   observations to a company instead of typing a name, which is what makes cross-project
   supplier analysis possible later.

8. **Create phases (optional).** `phasesCreatePhase` — `POST /projects/{project_id}/phases` — one
   per milestone. Observations and form instances can then be filed against a phase.

9. **Verify.** `projectDetailsCheckProjectSettings` — `GET /projects/{project_id}` — and confirm
   `member_count`, `status`, `is_active` and the dates read back as intended.

## Failure handling

Finalcad has **no idempotency key**. If step 6 or 7 fails, read the `api_code` in the error body
before retrying:

- `…_ERR{n}` — the endpoint rolled back. Nothing landed. Safe to replay the whole call.
- `…_WRN{n}` — the endpoint could **not** roll back. Some members or companies were created.
  Replaying will duplicate them; re-read the current list first
  (`projectMembersManagementGetMembers`, `companiesGetProjectCompaniesList`) and send only the
  difference.
- **HTTP 206** on a bulk call means partial success by design, not an error to retry blindly.

Error body shape is `{"statut", "api_code", "message", "data"}` — **not** RFC 9457 problem+json.
A 401 or 403 comes from the AWS gateway instead and has no `api_code` at all, so handle both shapes.

## Notes

- Do not create the project twice on a timeout. There is no way to ask "did my last POST land?"
  other than listing projects and matching on `case_number` — which is another reason to always
  send it.
- `projectDetailsDeleteAProject` has a `projectDetailsRestoreAProject` counterpart, so a mistaken
  project is recoverable. Most other deletes in this API are not.
