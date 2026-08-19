# Interfy API: five-minute quickstart

This guide uses the central production API. Every URL combines the server
`https://api.interfy.io` with the complete operation path shown in the OpenAPI
document, including `/api/v2`.

## 1. Set your workspace and credentials

The central API requires `X-Workspace` on every workspace-scoped request. The
identifier normally combines the domain family and workspace subdomain. For
example, `dev.interfy.io` uses `interfy-dev`. Confirm the production value with
your Interfy administrator.

```bash
export INTERFY_API_BASE="https://api.interfy.io"
export INTERFY_WORKSPACE="interfy-dev"
export INTERFY_LOGIN="you@example.com"
export INTERFY_PASSWORD="replace-me"
```

Keep credentials in a secret manager in real integrations; do not commit them
to source control.

## 2. Get a JWT

`login` accepts either an email address or username. A successful password-only
login returns the JWT in `token`.

```bash
LOGIN_RESPONSE="$(curl --fail-with-body --silent --show-error \
  --request POST \
  --url "${INTERFY_API_BASE}/api/v2/login" \
  --header "Content-Type: application/json" \
  --header "X-Workspace: ${INTERFY_WORKSPACE}" \
  --data "$(jq -n \
    --arg login "${INTERFY_LOGIN}" \
    --arg password "${INTERFY_PASSWORD}" \
    '{login: $login, password: $password}')")"

export INTERFY_TOKEN="$(printf '%s' "${LOGIN_RESPONSE}" | jq -r '.token // empty')"
test -n "${INTERFY_TOKEN}" || {
  printf 'Login did not return a token. Response: %s\n' "${LOGIN_RESPONSE}" >&2
  exit 1
}
```

Some workspaces require two-factor authentication or a password change. In
that case the response contains `requires_2fa`, `requires_2fa_setup`, or
`requires_password_change` instead of a token; complete that workspace's login
flow before calling protected endpoints.

For server-to-server integrations, Interfy also supports OAuth 2.0 at
`POST /oauth/token`. Ask your Interfy administrator for the correct client,
grant type, and scopes instead of embedding a user's password in a service.

## 3. Make an authenticated request

List ECM folders visible to the authenticated user:

```bash
curl --fail-with-body --silent --show-error \
  --url "${INTERFY_API_BASE}/api/v2/ecm/folders" \
  --header "Accept: application/json" \
  --header "Authorization: Bearer ${INTERFY_TOKEN}" \
  --header "X-Workspace: ${INTERFY_WORKSPACE}" | jq
```

List BPM process definitions visible to the user:

```bash
curl --fail-with-body --silent --show-error \
  --url "${INTERFY_API_BASE}/api/v2/bpm/processes" \
  --header "Accept: application/json" \
  --header "Authorization: Bearer ${INTERFY_TOKEN}" \
  --header "X-Workspace: ${INTERFY_WORKSPACE}" | jq
```

## 4. Create an ECM document

Use a folder `_id` returned by the folders endpoint. `template_id` and `fields`
are optional unless the target folder's configuration requires them.

```bash
export INTERFY_FOLDER_ID="replace-with-folder-id"

curl --fail-with-body --silent --show-error \
  --request POST \
  --url "${INTERFY_API_BASE}/api/v2/ecm/documents" \
  --header "Accept: application/json" \
  --header "Content-Type: application/json" \
  --header "Authorization: Bearer ${INTERFY_TOKEN}" \
  --header "X-Workspace: ${INTERFY_WORKSPACE}" \
  --data "$(jq -n \
    --arg folder_id "${INTERFY_FOLDER_ID}" \
    --arg title "Created through the Interfy API" \
    '{folder_id: $folder_id, title: $title}')" | jq
```

## 5. Upload a file, then attach it

Uploads are direct-to-storage. First request a presigned upload using the JWT
storage route:

```bash
export INTERFY_FILE="./example.pdf"

curl --fail-with-body --silent --show-error \
  --request POST \
  --url "${INTERFY_API_BASE}/api/v2/jwt/storage/presignedUpload" \
  --header "Accept: application/json" \
  --header "Content-Type: application/json" \
  --header "Authorization: Bearer ${INTERFY_TOKEN}" \
  --header "X-Workspace: ${INTERFY_WORKSPACE}" \
  --data "$(jq -n \
    --arg filename "$(basename "${INTERFY_FILE}")" \
    --arg content_type "application/pdf" \
    --argjson file_size "$(wc -c < "${INTERFY_FILE}" | tr -d ' ')" \
    '{filename: $filename, content_type: $content_type, extension: "pdf", file_size: $file_size}')" | jq
```

Upload the bytes using the returned URL and fields, then pass the returned
`remote_path`, file name, MIME type, and byte size in the `files` array of
`POST /api/v2/ecm/documents`. Use the multipart endpoints under
`/api/v2/jwt/storage/multipartUpload` for large files. Presigned responses can
vary by storage provider, so follow the HTTP method and form fields returned by
your workspace rather than assuming a fixed S3 request shape.

## Pagination and errors

- Pagination parameters vary by endpoint. Where documented, start with
  `?page=1&per_page=50` and follow the response's pagination metadata or links.
- `401` means the token is absent, invalid, or expired.
- `403` means the identity is authenticated but lacks permission, a workspace
  policy blocks the operation, or a capacity limit was reached.
- `404` can mean the resource does not exist or is not visible in the selected
  workspace. Recheck both the ID and `X-Workspace`.
- `422` is a validation failure. Read the JSON `errors` object for field-level
  messages.
- `429` means a rate limit was reached. Respect `Retry-After` when present and
  use exponential backoff with jitter.

Never retry mutating requests blindly. Use a stable application-side operation
ID and reconcile the resource state before retrying after a timeout.
