---
name: Manage loan officers, team members and the organization hierarchy
description: Create and maintain loan officers, team members, partners, organization nodes and state licenses, including the polymorphic actions endpoint that carries promote, demote, enable and disable.
api: openapi/ncino-mortgage-openapi.yml
operations: [loan_officers-index, loan_officers-show, loan_officers-create, loan_officers-update, loan_officers-actions, team_members-index, team_members-create, team_members-update, team_members-actions, companies-index, companies-show, company_regions-index, company_regions-create, region_branches-index, region_branches-create, loan_officer_state_licenses-index, loan_officer_state_licenses-create, loan_officer_partners-index, loan_officer_partners-create, loan_officer_partners-actions, loan_officer_assignments-index, loan_officer_assignments-create]
---

# Manage loan officers, team members and the organization hierarchy

Authenticate first — see `ncino-authenticate-and-call.md`.

## The hierarchy

nCino models three organization levels: **Company → Region → Branch**. Read the company
with `companies-index` / `companies-show`, its regions with `company_regions-index`, and
a region's branches with `region_branches-index`. Create with `company_regions-create`
and `region_branches-create`. The MCP surface flattens all of this into one
"organization" concept; the REST surface does not.

## Users

- Loan officers: `loan_officers-index`, `loan_officers-show`, `loan_officers-create`,
  `loan_officers-update`.
- Team members: `team_members-index`, `team_members-create`, `team_members-update`.
- Partners for a loan officer: `loan_officer_partners-index`, `-show`, `-create`, `-update`.

Creating a user with an email that already exists returns **409**, not a merge. Search
with the `email` query filter on the `-index` operation before creating.

## State transitions go through `-actions`

Promote, demote, enable, disable, transfer between orgs, and send-welcome-email are all
action values on the polymorphic actions operation — `loan_officers-actions`
(`POST /users/loan_officers/{loan_officer_id}/actions`), `team_members-actions`,
`loan_officer_partners-actions`. Read the request schema in the spec for the exact
action name; do not invent one.

These are consequential, irreversible-feeling operations on real people's access.
Confirm with the operator before promoting to admin or disabling a user. Disabling a
loan officer can reassign their borrowers and partners.

While one of these is in flight, a second write may return **423 Locked** ("Action
cannot be performed due to an ongoing loan officer operation"). Back off and re-poll.

## Assignments and licensing

`loan_officer_assignments-index` / `-create` bind a user to an organization node;
`loan_officer_assignments-index_available` lists what they may be assigned to.
State licensing is tracked separately at every level:
`loan_officer_state_licenses-*`, `company_state_licenses-*`, `region_state_licenses-*`,
`branch_state_licenses-*`, with company-level templates at
`company_state_license_templates-*`.

## Confirm the effect

Every one of these transitions has a matching webhook event — `loan_officer_promoted`,
`loan_officer_demoted`, `user_enabled`, `user_disabled`, `organization_created`,
`state_license_updated`. See `asyncapi/ncino-mortgage-webhooks.yml`.
