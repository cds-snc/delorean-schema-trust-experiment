````yaml
# Feature Implementation Schema
# Governs: the artifact checklist an agent must complete before marking a slice done.
# An agent uses this to self-check before submitting for review.

version: 1
artifact: feature

# An agent must confirm each item before the slice is complete.
# Record actual status in the compliance log.

checklist:
  backend:
    - id: pydantic_request_model
      description: A Pydantic model governs the request shape. ConfigDict(extra="forbid") is set.
      required: true

    - id: pydantic_response_model
      description: A Pydantic model governs the response shape.
      required: true

    - id: router_uses_response_model
      description: The FastAPI router declares response_model on every route.
      required: true

    - id: service_layer
      description: Business logic lives in a service, not the router.
      required: true

    - id: http_status_codes_explicit
      description: All non-200 status codes are declared explicitly on the route.
      required: true

    - id: tests_added
      description: At least one happy-path and one error-path test exist for the new route.
      required: true

  frontend:
    - id: typescript_interface
      description: A TypeScript interface mirrors the backend response model.
      required: true

    - id: api_client_typed
      description: The API client function is typed with the interface.
      required: true

    - id: error_state_handled
      description: The UI handles and displays error states from the API.
      required: true

    - id: loading_state_handled
      description: The UI shows a loading state during async operations.
      required: true

  api_contract:
    - id: openapi_exported
      description: make export-openapi ran and the openapi/openapi.json is updated.
      required: true

    - id: openapi_check_passes
      description: make check-openapi passes with no diff.
      required: true

  general:
    - id: no_raw_sql
      description: No raw SQL strings in route handlers or services.
      required: true

    - id: no_secrets_in_code
      description: No credentials, tokens, or real environment values in committed files.
      required: true

validation_rules:
  - rule: all required checklist items must be confirmed before slice is marked complete
  - rule: openapi_check_passes must be confirmed last — it validates the whole contract
  - rule: if any item is false, the agent must fix it or record a waiver with justification
````