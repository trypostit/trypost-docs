# TryPost Documentation

Docs for [TryPost](https://trypost.it) — open-source social media scheduling platform. Built with [Mintlify](https://mintlify.com).

## Project Context

TryPost is a Laravel + Inertia.js + **Vue 3** application that lets users schedule and publish posts across 10 social platforms (12 platform identifiers in code, since LinkedIn and Instagram each have two connection flavors). It has two deployment modes: **Cloud** (managed by us) and **Self-Hosted** (user deploys on their own server).

The app source code lives at `~/Herd/trypost`. This repo (`trypost-docs`) is the documentation site only.

## URL Architecture

**CRITICAL: There are NO subdomains. Everything uses path-based routing under a single domain.**

### Cloud
| Service | URL |
|---------|-----|
| Web app | `https://app.trypost.it` |
| REST API | `https://app.trypost.it/api` |
| MCP server | `https://app.trypost.it/mcp/trypost` |
| Register | `https://app.trypost.it/register` |

### Self-Hosted
Same paths, user configures `APP_URL` in `.env`:
| Service | URL |
|---------|-----|
| Web app | `{APP_URL}` |
| REST API | `{APP_URL}/api` |
| MCP server | `{APP_URL}/mcp/trypost` |

**Never use `api.trypost.it` or `mcp.trypost.it` — these do not exist.**

## Documentation Structure

5 top-level navigation anchors in `docs.json`:

| Anchor | Icon | Purpose |
|--------|------|---------|
| **Documentation** | `book-open` | Product docs: quickstart, platforms, features, contributing |
| **API Reference** | `code` | REST API endpoint docs with curl examples |
| **Build with AI** | `microchip-ai` | MCP server intro, tools reference, 10 setup guides |
| **Knowledge Base** | `book` | Conceptual guides: posts, media, signatures, labels, accounts, team, notifications |
| **Self-Hosting** | `server` | Installation, configuration, production, Docker — isolated from the rest |

## Writing Rules

### Cloud-first
- Default instructions assume the user is on **TryPost Cloud**
- Self-hosted specifics go in `<Accordion>`, `<Note>`, or the Self-Hosting anchor
- Example: Platform pages say "click Connect" for cloud, API credentials in `<Accordion>` for self-hosted

### URLs in examples
- API curl examples: `https://app.trypost.it/api/...`
- MCP server configs: `https://app.trypost.it/mcp/trypost`
- Dashboard links: `https://app.trypost.it`
- Cloud media URLs (R2 with custom domain): `https://media.trypost.it/medias/{uuid}.{ext}`
- Self-hosted examples: `{APP_URL}/api/...` or `https://trypost.yourdomain.com/api/...`
- Stored media `path` is always `medias/{uuid}.{ext}`; the public URL comes from `Storage::url($path)` which prepends the configured disk URL (`R2_URL`, S3 endpoint, or local `${APP_URL}/storage`).

### Affiliate & partner links
- **SendKit** (default mailer): `https://sendkit.dev?utm_source=trypost&utm_medium=docs&utm_campaign={context}`
- **Hetzner** (recommended hosting): `https://hetzner.cloud/?ref=V4djx1Vt7Mm7` — always disclose as referral link

## API Reference

### Authentication
- **REST API**: Bearer token in `Authorization` header (Personal Access Tokens issued by Laravel Passport, JWT format — do not assume any prefix or fixed length). In examples use `YOUR_API_KEY` as the placeholder.
- **MCP server**: OAuth 2.1 with Dynamic Client Registration (PKCE S256, scope `mcp:use`). Clients discover endpoints via `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource`, register at `/oauth/register`, then walk the user through `/oauth/authorize` → `/oauth/token` in a browser. The same PAT also works as a Bearer token against the MCP endpoint as a fallback for CLIs that don't speak OAuth yet.
- Same PAT works for REST API and MCP server (Bearer fallback).
- Issued via `POST /api/api-keys` or **Settings > API Keys** in the dashboard.

### Endpoints
| Group | Endpoints |
|-------|-----------|
| Posts | `GET /posts`, `POST /posts`, `GET /posts/{post}`, `PUT /posts/{post}`, `DELETE /posts/{post}`, `POST /posts/{post}/media`, `POST /posts/{post}/media/from-url`, `GET /posts/{post}/metrics`, `GET /posts/{post}/preview` |
| Platforms | `GET /content-types` |
| Workspace | `GET /workspace` |
| Signatures | `GET /signatures`, `POST /signatures`, `PUT /signatures/{id}`, `DELETE /signatures/{id}` |
| Labels | `GET /labels`, `POST /labels`, `PUT /labels/{id}`, `DELETE /labels/{id}` |
| Social Accounts | `GET /social-accounts`, `PUT /social-accounts/{account}/toggle` |
| API Keys | `GET /api-keys`, `POST /api-keys`, `DELETE /api-keys/{apiToken}` |

`PUT /posts/{post}` accepts a `status` of `publishing` to publish immediately (no separate publish endpoint at the REST level).

### Response shape
- Resources are returned **unwrapped** (`JsonResource::withoutWrapping()` is set globally). A single resource looks like `{ "id": ..., ... }` — no `data:` envelope.
- Non-paginated collections return a plain JSON array (`[ {...}, {...} ]`).
- Only `GET /posts` returns the Laravel pagination envelope (`{ data, links, meta }`).
- `attach-media-from-url`, `metrics`, and `preview` are bespoke — not Resource-shaped (no `data:` wrapper, just the documented fields at top level). The multipart `/media` upload returns a regular Post resource.
- `POST /api-keys` returns `{ "token": {...}, "plain_token": "..." }`. The plain token is shown ONCE.

### Pagination
Only `GET /posts` paginates (15 per page). All other list endpoints return full results as a plain array.

## MCP Server

- Tool names are auto-derived from class basenames as kebab-case + `-tool` suffix (Laravel MCP convention — no `#[Name]` overrides in `TryPostServer`).
- Post tools: `list-posts-tool`, `get-post-tool`, `create-post-tool`, `update-post-tool`, `publish-post-tool`, `preview-post-tool`, `delete-post-tool`, `attach-media-from-url-tool`, `get-post-metrics-tool`.
- Plus: `list-content-types-tool` (Platforms), `list-signatures-tool` / `create-signature-tool` / `update-signature-tool` / `delete-signature-tool`, `list-labels-tool` / `create-label-tool` / `update-label-tool` / `delete-label-tool`, `list-social-accounts-tool` / `toggle-social-account-tool`, `get-workspace-tool`, `list-api-keys-tool` / `create-api-key-tool` / `delete-api-key-tool`.
- `create-post-tool` accepts `platforms[]` (each with `social_account_id` and `content_type`), `scheduled_at`, `label_ids`, etc. — same shape as REST `POST /posts`.
- `publish-post-tool` is a separate, destructive tool (annotated `IsDestructive`); REST clients use `PUT /posts/{id}` with `status=publishing` instead.
- Server route: `Mcp::web('/mcp/trypost', TryPostServer::class)->middleware(['auth:api', 'workspace.token'])`. The `workspace.token` middleware (`LoadWorkspaceFromToken`) is what returns `402 Active subscription required` on Cloud accounts without an active trial/subscription.

## Enum Values

These are the exact string values used in API responses. Never use alternatives.

### Post status
`draft`, `scheduled`, `publishing`, `published`, `partially_published`, `failed`

### PostPlatform status
`pending`, `publishing`, `published`, `failed`

### Social account status
`connected`, `disconnected`, `token_expired`

**NOT `active`/`inactive` — those are for `is_active` boolean field.**

### Social account platforms (12 values)
`linkedin`, `linkedin-page`, `x`, `tiktok`, `youtube`, `facebook`, `instagram`, `instagram-facebook`, `threads`, `pinterest`, `bluesky`, `mastodon`

`instagram` is the standalone Basic Display flow; `instagram-facebook` is the Business-via-Facebook-Page flavor.

### Content types (21 values)
| Platform | Content types |
|----------|--------------|
| LinkedIn | `linkedin_post`, `linkedin_carousel` |
| LinkedIn Page | `linkedin_page_post`, `linkedin_page_carousel` |
| X | `x_post` |
| Facebook | `facebook_post`, `facebook_reel`, `facebook_story` |
| Instagram | `instagram_feed`, `instagram_carousel`, `instagram_reel`, `instagram_story` |
| TikTok | `tiktok_video`, `tiktok_photo` |
| YouTube | `youtube_short` |
| Threads | `threads_post` |
| Pinterest | `pinterest_pin`, `pinterest_video_pin`, `pinterest_carousel` |
| Bluesky | `bluesky_post` |
| Mastodon | `mastodon_post` |

Instagram content types work for both `instagram` and `instagram-facebook` accounts.

**Never use generic values like `text`, `image`, `video`.**

### API token state
There is no `status` enum on tokens. `AccessToken` exposes `revoked` (boolean) and `expires_at` (nullable timestamp). The `ApiKeyResource` derives `is_active` from `! revoked && (expires_at is null || expires_at > now)`.

### Media types (2 values)
`image`, `video`

There is no `document` type. Allowed MIME types: images = `image/jpeg`, `image/png`, `image/gif`, `image/webp` (PNG/WebP stills are normalized to JPEG q100); videos = `video/mp4`, `video/quicktime`. Default size caps: 10 MB image, 1024 MB video (overridable via `MEDIA_IMAGE_MAX_SIZE_MB` / `MEDIA_VIDEO_MAX_SIZE_MB`).

### Notification types (9 values)
`post_published`, `post_failed`, `post_partially_published`, `post_ready`, `account_disconnected`, `invite_received`, `member_joined`, `member_removed`, `mentioned_in_comment`

### Notification channels
`email`, `in_app`, `both`

### Workspace roles (3 values)
`admin`, `member`, `viewer`

There is no `owner` role — the workspace owner is the user on `workspaces.owner_id`. Roles only apply to additional members joined via invites.

## Emails Sent by TryPost

| Email | Trigger |
|-------|---------|
| `PostPublished` | Post published successfully |
| `PostPublishFailed` | Post failed to publish on one or more platforms |
| `AccountDisconnected` | A single social account lost authorization |
| `WorkspaceConnectionsDisconnected` | Daily check found disconnected accounts in a workspace |
| `WorkspaceInvite` | Team member invited to join a workspace |
| Email verification | New user registration (Laravel built-in) |
| Password reset | User requests password reset (Laravel built-in) |

Default mailer is **SendKit** (`MAIL_MAILER=sendkit`).

## Scheduled Commands

| Command | Schedule | Purpose |
|---------|----------|---------|
| `posts:process-scheduled` | Every minute | Dispatches `PublishPost` jobs for posts that are due |
| `social:refresh-expiring-tokens` | Hourly | Proactively refreshes OAuth tokens before they expire |
| `social:recover-stuck-posts` | Every 30 minutes | Recovers posts stuck in `publishing` (retry or mark failed) |
| `social:check-connections` | Daily | Verifies all connected social accounts still authenticate |

## Self-Hosted Requirements

- PHP 8.2+, Node.js 20+, PostgreSQL 14+ or MySQL 8+, Redis 6+
- Supervisor for Horizon (queues) and Reverb (WebSockets)
- Cron job running `php artisan schedule:run` every minute
- Upload limits: `upload_max_filesize=2G`, `post_max_size=2G` (video support)
- Bluesky and Mastodon work without API credentials
- All other platforms require developer app credentials

## Supported Languages

English (`en`), Spanish (`es`), Portuguese (`pt-BR`).

## Running Docs Locally

```bash
npx @mintlify/cli dev
```
