---
name: Create a loan and attach borrowers
description: Walk a loan from creation through borrower attachment, milestone reads and document tasks using the nCino Mortgage REST operations, with the beta and retry caveats that apply.
api: openapi/ncino-mortgage-openapi.yml
operations: [loans-create, loans-show, loans-update, loans-index, loan_borrowers-create, loan_borrowers-index, loan_borrowers-show, loan_borrowers-update, loan_milestones-index, loan_doc_tasks-create, loan_doc_tasks-index, loan_documents-index, loans-actions, jobs-show]
---

# Create a loan and attach borrowers

Authenticate first — see `ncino-authenticate-and-call.md`.

**Read this before writing anything:** most operations in this flow are marked beta by
nCino ("functionality may change without notice, support is limited, and use in
production is not recommended"). There is also **no idempotency key** on this API. If
a `POST` times out, do not blindly retry it — read back with the matching `-index`
operation first, or you will create a duplicate. nCino reports duplicates as `409`
after the fact, not as a replayed success.

## 1. Create the loan

`loans-create` — `POST /loans`. Supply the loan officer, loan number, type, purpose,
amount, dates, rates and property information.

## 2. Add borrowers

`loan_borrowers-create` — `POST /loans/{loan_id}/borrowers` — for each borrower.
Borrower bodies carry PII (name, DOB, SSN, address). Do not echo SSN or DOB back into
conversation, logs, or downstream tool calls. Read back with `loan_borrowers-index`
and confirm before adding another.

## 3. Read the loan back

`loans-show` — `GET /loans/{loan_id}`. Use `loans-index` with `page` / `page_size` and
the `created_after` / `updated_after` filters to page a working set; there is no
cursor and no Link header.

## 4. Track progress

`loan_milestones-index` — `GET /loans/{loan_id}/loan_milestones` — for milestone state.
`loan_documents-index` and `loan_doc_tasks-index` for the document surface;
`loan_doc_tasks-create` to request a document from a borrower.

## 5. Change loan state

State transitions are not separate verbs. `loans-actions` — `POST /loans/{loan_id}/actions`
— takes the action in the body (activate, deactivate, convert, and similar). Consult the
request schema in the spec rather than guessing an action name.

## 6. Asynchronous work

Some actions return a job rather than a result. Poll `jobs-show` (`GET /jobs/{job_id}`)
or subscribe to the `job_status_changed` webhook. While a job is running, related
writes return `409` ("Action cannot be performed due to an ongoing asynchronous job")
or `423` — back off and re-poll rather than retrying immediately.
