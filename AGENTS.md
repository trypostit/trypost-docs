# TryPost Documentation

## Agent instructions

`AGENTS.md` is the single source of repository instructions for all coding agents. Read and follow this file before working on the documentation.

- Mintlify site: pages are MDX with YAML frontmatter; navigation, `openapi` and `redirects` live in `docs.json`.
- Preview with `mint dev`; check with `npx mint validate` and `npx mint broken-links`.
- Never rename or remove the frozen URLs listed below; every moved or removed slug gets a `docs.json` redirect.
- Every factual claim must trace to the app at `~/Herd/trypost` (cite `file:line` in the PR).
- Use active voice, second person, sentence-case headings, bold for UI labels exactly as in `lang/en/*.php`.

Docs for [TryPost](https://trypost.it) — open-source social media scheduling platform. Built with [Mintlify](https://mintlify.com).

## Project Context

TryPost is a Laravel + Inertia.js + **Vue 3** application that lets users plan, schedule and publish posts across 13 social networks (15 platform identifiers in code, since LinkedIn and Instagram each have two connection flavors), and see their performance in Insights. It has two deployment modes: **Cloud** (managed by us) and **Self-Hosted** (user deploys on their own server).

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

5 top-level tabs in `docs.json` (`navigation.tabs`):

| Tab | Icon | Purpose |
|-----|------|---------|
| **Documentation** | `book-open` | Home (`index`), quickstart, one page per network under `platforms/`, community pages |
| **API Reference** | `code` | REST API. Operation pages are generated from **`openapi.json`** at the repo root (hand-written from the app's `routes/api.php`, `app/Http/Requests/Api/**`, `app/Http/Resources/**`). Prose that does not fit one operation lives in `api-reference/introduction.mdx` and `api-reference/guides/*.mdx`. There are no per-endpoint `.mdx` files any more; old `api-reference/endpoint/*` slugs redirect |
| **Build with AI** | `microchip-ai` | MCP server intro, tools reference, setup guides per client |
| **Knowledge Base** | `book` | How the app works. Groups mirror the app sidebar, then the settings sidebar: Getting started, Create, Publish, Insights, Repurpose, Channels, Settings (Personal, Workspace, Content, Integrations). Page titles use the on-screen labels from `lang/en/*.php` |
| **Self-Hosting** | `server` | Overview, requirements, installation, configuration, AI providers, Docker, production, upgrading — isolated from the rest |

Every removed or moved slug gets an entry in `docs.json` `redirects`.

### Platform pages

`platforms/<slug>.mdx`, one per network: `linkedin`, `x-twitter`, `facebook`, `instagram`, `tiktok`, `youtube`, `threads`, `pinterest`, `bluesky`, `mastodon`, `telegram`, `discord`, `google-business`. Each follows the same layout: Connect, Supported content types, Media limits, Text and per-post options, Insights, OAuth scopes, Self-hosting setup (credentials inside `<Accordion>`).

### Frozen URLs (the app links to them — never rename or remove)

- `/platforms/<slug>` for every page above (`resources/js/lib/docs.ts` `platformGuideDocsUrl`)
- `/knowledge-base/media` and its per-network anchors `#instagram`, `#facebook`, `#threads`, `#x-twitter`, `#linkedin`, `#tiktok`, `#youtube`, `#pinterest`, `#bluesky`, `#mastodon`, `#discord`, `#telegram`, `#google-business-profile` (`mediaLimitsDocsUrl`)
- `/ai/introduction` and the site root

If one of these must move, add a redirect and change the app in the same release.

## Writing Rules

### Cloud-first
- Default instructions assume the user is on **TryPost Cloud**
- Self-hosted specifics go in `<Accordion>`, `<Note>`, or the Self-Hosting tab
- Example: Platform pages say "click Connect" for cloud, API credentials in `<Accordion>` for self-hosted
- This includes `api-reference/` — those endpoints are documented against `https://app.trypost.it/api`, so describe the Cloud response

**When a fixed behaviour becomes configurable**, keep the Cloud value as the statement and push the setting into self-hosting. Do not rewrite body prose into a two-mode comparison — it makes every reader parse a branch that only one of them is on.

```mdx
{/* Wrong — the Cloud reader has to work out which half applies to them */}
On TryPost Cloud you can connect several accounts on the same network. On
self-hosted instances that used to be a setting.

{/* Right — Cloud is the statement, self-hosting is a pointer */}
A workspace can connect as many accounts as you want on the same network.

<Note>
  Self-hosting TryPost? Same rule — see
  [multiple accounts per network](/self-hosting/configuration#multiple-accounts-per-network).
</Note>
```

The same applies to anything a self-hoster can switch off or raise: platform availability, limits, AI providers. State what Cloud does; link out for the knob.

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
- **REST API**: Bearer token in the `Authorization` header (Personal Access Tokens issued by Laravel Passport, JWT format — do not assume any prefix or fixed length). In examples use `YOUR_API_KEY` as the placeholder.
- **MCP server**: OAuth 2.1 with Dynamic Client Registration (PKCE S256, scope `mcp:use`). Clients discover endpoints via `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource`, register at `/oauth/register`, then walk the user through `/oauth/authorize` → `/oauth/token` in a browser. Personal Access Tokens are **not** accepted on `/mcp/trypost` — use the REST API for scripts/CI that cannot complete OAuth.
- Issued via `POST /api/api-keys` or **Settings → API** in the dashboard (page **API Keys**).

### Endpoints, shapes and pagination

`openapi.json` is the source of truth for every operation, request field, response shape and error (hand-written from the app's `routes/api.php`, `app/Http/Requests/Api/**`, `app/Http/Resources/**`, `app/Http/Controllers/Api/**`). Public API = routes whose URI starts with `api/` (89 today: `php artisan route:list --path=api`). Edit the spec, not generated pages. `api-reference/introduction.mdx` covers auth, errors, pagination, rate limits and timestamps; `api-reference/guides/posting.mdx` covers `meta` per network, scheduling modes and thread replies; `api-reference/guides/media-uploads.mdx` covers media. Update those, not this file. Facts to keep in mind:

- Resources are unwrapped (`JsonResource::withoutWrapping()`); lists use the Laravel envelope `{ data, links, meta }` with the page size from `config('app.pagination.default')` (25 today). Always tell readers to read `meta.per_page`.
- `PUT /posts/{post}` with `status: publishing` publishes now; there is no separate publish endpoint.
- Post `meta` rules live only in the app's `App\Support\PostPlatformMetaRules`; document them from there. Unknown keys are dropped; on update `meta` is merged and `null` clears a nullable key. Boolean meta keys do not accept `null`; send `true` or `false`.
- There is no asset-library endpoint and no account toggle in the REST API.
- Enum values (statuses, content types, webhook events, platforms) are documented in `openapi.json` schemas; copy them from the app enums under `app/Enums/**`, never from memory.

## MCP Server

- The tool list is exactly `TryPostServer::$tools` (`app/Mcp/Servers/TryPostServer.php`), 86 tools today; `ai/tools-reference.mdx` documents every one and `check_coverage.py mcp` enforces it. Names are the class basename in kebab-case plus `-tool`, except `get-tiktok-creator-info-tool` (`#[Name]`).
- `create-post-tool` / `create-posts-tool` / `update-post-tool` accept inline `media[]` (`url`, `upload_token` or `id`); `update-post-tool` cannot change the channel. `create-api-key-tool` returns `{token, plain_token}` like REST.
- Gates mirror the web: any member (`createPost`) for posts, notes, ideas, labels, signatures, analytics and channel reads; `publishDirectly` for approve/reject, set recurrence, reorder queue, move to slot and every repurpose tool; admins (`manageAccounts`, `manageWebhooks`, `manageTeam`) for posting-schedule writes, webhooks and API keys.
- Server route: `Mcp::web('/mcp/trypost', TryPostServer::class)->middleware(['auth:api', 'workspace.token:mcp'])`. `LoadWorkspaceFromToken` requires an active OAuth grant with `mcp:use` (`403 MCP OAuth authorization required.` otherwise), binds the token's workspace (`401 No workspace selected.`, `403 Workspace access denied.`), and returns `402 Active subscription required.` on Cloud without app access.
- There is no REST endpoint or MCP tool for generating or rewriting content with AI. Repurpose can shorten captions with AI in the background, including for repurposes created through REST or MCP (`app/Services/Repurpose/CaptionAdapter.php`).

## Notifications and permissions

### Notification types (email only)
`post_published`, `post_failed`, `account_disconnected`, `post_at_risk`, `post_note_added`, `collaboration` (approval requests and decisions). `post_ready` and `mentioned_in_comment` exist only so 1.x queued jobs finish; never document them. There is no in-app notification center and no notification channel setting: every notification is an email.

### Member permissions (no roles)
A membership (`user_workspace`) and an invite carry two flags: `is_admin` (manages members, settings, channels) and `requires_approval` (the member's posts need approval; always false for admins). The Admin / Member / Viewer roles were removed in 2.0. The account owner is always an admin and publishes directly. Every member can create posts.

## Emails

Every email the app sends is a Maizzle template in `maizzle/templates/` with a Mailable in `app/Mail/`; the self-hosting list is `self-hosting/configuration.mdx` → **Emails sent by TryPost**. Default mailer is SendKit (`config/mail.php:19`).

## Coverage checker

`python3 ~/Herd/trypost/docs/superpowers/audits/2026-10-07-docs-inventory/check_coverage.py all` must print `OK` (api, mcp, env, platforms, redirects).

## Scheduled Commands

The full table (17 entries, including analytics, retention and Google Business review checks) lives in `self-hosting/production.mdx` → **Scheduled Tasks (Cron)**; the source is `routes/console.php` / `php artisan schedule:list`.

## Self-Hosted Requirements

- PHP 8.4.1+ (the Docker image ships 8.4), Node.js 22.12+, PostgreSQL or MySQL, Redis
- Horizon (queues), Reverb (WebSockets) and the scheduler running continuously
- Imagick with HEIC for iPhone photo conversion (in the Docker image)
- Upload limits: `upload_max_filesize=1G`, `post_max_size=1G` (matches the Docker image)
- Bluesky and Mastodon work without API credentials; every other network needs developer app credentials (Google Business Profile has its own Google client, separate from YouTube)
- The docs describe the current `main` only (owner decision October 2026): no upgrade path from 1.x, no `release:trypost-2`, no "before/since 2.0" history. An upgrade is the routine in `self-hosting/upgrading.mdx` plus the GitHub release notes

## Supported Languages

17 locales (`App\Enums\User\Locale`): `en`, `uk`, `pt-BR`, `es`, `fr`, `de`, `it`, `nl`, `pl`, `el`, `ja`, `ko`, `zh`, `ru`, `tr`, `ar`, `id`. Docs are written in English only.

## Running Docs Locally

```bash
npx @mintlify/cli dev
```
