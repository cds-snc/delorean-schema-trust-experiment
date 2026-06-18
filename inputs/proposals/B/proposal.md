# Proposal: Delete Example Item (Schema-Guided)

**Change ID:** delete-schema-guided  
**Approach:** Schema-guided execution. Require schema validation before advancing stages.

This is the B side of the A/B demo. The feature brief is intentionally the same
as the unguided proposal so the comparison isolates one variable: schema
contract usage.

---

## Feature Brief

**title:** Delete an example item

**capability:** example-items

**work_context:** local

**problem:**  
A developer using the data example page can create example items but cannot
remove them without resetting the database. This makes it hard to clean up
test data during development and makes the UI feel incomplete as a starter
example of a full CRUD path.

**scenarios:**

- **Scenario 1 — Happy path (item deleted)**
  - Given: The data example page is open and at least one item exists
  - When: The developer clicks the Delete button on an item and confirms
  - Then: The item is removed from the list, the database row is soft-deleted,
    and a success confirmation message is shown

- **Scenario 2 — Deletion of non-existent item**
  - Given: The developer tries to delete an item that no longer exists
  - When: The DELETE request reaches the backend
  - Then: The API returns 404 with a structured error message; the frontend
    shows a user-visible error state

- **Scenario 3 — Confirmation prevents accidental deletion**
  - Given: The developer clicks Delete
  - When: They dismiss the confirmation without confirming
  - Then: No deletion occurs and the item remains in the list

**out_of_scope:**
  - Bulk deletion
  - Undo / restore deleted items
  - Authorization or role-based access control
  - Production data deletion flows

**impacted_artifacts:**
  - `backend/app/routers/example_items.py` - new DELETE route
  - `backend/app/models/example_items.py` - no model changes expected; verify
  - `backend/app/services/example_item_service.py` - soft-delete method
  - `backend/app/repositories/example_item_repository.py` - delete query
  - `backend/tests/` - new route test and service test
  - `frontend/src/features/example-data/api/exampleItems.ts` - deleteExampleItem function
  - `frontend/src/features/example-data/pages/ExampleDataPage.tsx` - delete button + confirmation
  - `openapi/openapi.json` - updated after make export-openapi
  - `delorean/evidence/delete-schema-guided/compliance-log.yaml` - populated during build

---

## Execution Instruction (A/B Variable)

Require schema validation while producing artifacts for this change:

1. Validate the proposal against `delorean/schemas/prd.schema.yaml`.
2. Validate implementation completion against `delorean/schemas/feature.schema.yaml`.
3. Validate evidence summary against `delorean/schemas/evidence.schema.yaml`.
4. Log each validation event in `delorean/evidence/delete-schema-guided/compliance-log.yaml`.
