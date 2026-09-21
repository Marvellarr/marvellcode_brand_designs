# MarvelCode Site Architecture

**Status:** Proposed baseline architecture  
**Repository:** `Marvellarr/marvellcode_brand_designs`  
**Audience:** Product, design, and engineering collaborators

## 1. Executive Summary

The repository currently contains the MarvelCode visual system, reference imagery, and several standalone HTML mockups. It does not yet contain a production application, build pipeline, shared component library, or persistent backend. The recommended architecture turns those assets into one coherent web product with three clearly separated surfaces:

1. **Public site:** brand story, capabilities, platform overview, analytics, security, documentation, and contact/conversion paths.
2. **Identity surface:** sign-in, account creation, password recovery, and legal/support links.
3. **Authenticated platform:** a dashboard for pipeline telemetry, deployments, AI models, cloud tooling, analytics, and workspace settings.

The public and identity surfaces should share the same design system and application shell. The authenticated platform should share tokens and primitives, but use a dedicated application shell with a collapsible utility rail and workspace canvas. This prevents marketing navigation from becoming coupled to operational dashboard navigation.

## 2. Current-State Assessment

The repository is public, uses `main` as its only branch, and is currently an HTML/design archive. The main materials are under `stitch_brand_website_and_screen_design/` and include brand references, design specifications, and page mockups for the landing page, platform portal, sign-in, and account creation flows. There is no `package.json`, framework configuration, application entry point, test suite, or GitHub Actions workflow yet.

The existing design direction is strong and consistent: a dark cyber-intelligent enterprise aesthetic, electric blue-to-violet accents, Inter for interface text, Montserrat for display headings, JetBrains Mono for technical identifiers and telemetry, a 12-column desktop grid, glass-like layered surfaces, and a utility rail for complex tools.

## 3. Target Information Architecture

### Public routes

| Route | Purpose | Primary action |
|---|---|---|
| `/` | Brand promise, capability summary, proof points, platform teaser | Explore platform / Start a conversation |
| `/solutions` | Detailed capability suites | View a capability or contact MarvelCode |
| `/platform` | Product/platform overview | Sign in or request access |
| `/platform/technology` | Architecture, infrastructure, and technology narrative | Read documentation |
| `/analytics` | Business intelligence and neural-data offering | Explore analytics |
| `/brand-system` | Public brand/design system reference if desired | View brand system |
| `/security` | Security posture, controls, whitepaper access | Read security material |
| `/documentation` | Public developer and product documentation | Open a guide |
| `/status` | Service availability and incidents | View current status |
| `/contact` | Lead/contact form | Submit an inquiry |

### Identity routes

| Route | Purpose |
|---|---|
| `/login` | Sign in to MarvelCode |
| `/create-account` | Create a user account |
| `/forgot-password` | Start password recovery |
| `/verify-email` | Complete email verification |
| `/terms` | Terms of service |
| `/privacy` | Privacy policy |

### Authenticated platform routes

| Route | Purpose |
|---|---|
| `/app` | Redirect to the user's default workspace |
| `/app/overview` | Pipeline overview and system health |
| `/app/deployments` | Deployment history, status, and detail |
| `/app/services` | Microservices health and runtime signals |
| `/app/models` | AI model registry and recent model activity |
| `/app/analytics` | Workspace analytics and reports |
| `/app/toolkit` | Cloud actions and developer utilities |
| `/app/docs` | Contextual product/developer documentation |
| `/app/settings` | Profile, workspace, access, API keys, notifications |

Unknown routes should resolve to a branded not-found page. All `/app/*` routes must be protected by authentication and workspace authorization.

## 4. Recommended Application Shape

Use a single TypeScript web application with route-level separation rather than multiple disconnected HTML pages. A React-based framework with server rendering and static generation support is appropriate; the implementation should use the repository's existing HTML as visual references, not as production page templates.

```text
src/
  app/                         # route entries and layouts
    (marketing)/               # public site shell and pages
    (auth)/                    # sign-in and account flows
    app/                       # authenticated platform shell
  components/
    ui/                        # buttons, inputs, cards, badges, dialogs
    marketing/                 # hero, capability grid, proof metrics, CTA
    platform/                  # KPI cards, telemetry, tables, command palette
    navigation/                # public nav, footer, rail, breadcrumbs
  design-system/
    tokens.ts                  # color, type, spacing, radius, elevation
    themes.ts                  # light/dark or future tenant themes
  features/
    auth/
    deployments/
    services/
    models/
    analytics/
    workspace/
  lib/
    auth/
    api/
    validation/
    telemetry/
  styles/
    globals.css
    utilities.css
  types/
    api.ts
    domain.ts
public/
  brand/                       # approved logos and reference exports
  icons/
  images/

content/
  docs/
  legal/

 tests/
   unit/
   integration/
   e2e/
```

The first implementation can use a static/mock data adapter so the screens become functional before a production API is available. Keep all data access behind feature services such as `deploymentService`, `workspaceService`, and `analyticsService`; components should not fetch directly from arbitrary endpoints.

## 5. Shells and Navigation

### Marketing shell

The marketing shell contains the MarvelCode wordmark, links to Solutions, Platform, Technology, Analytics, Documentation, Security, and Status, plus a primary sign-in or request-access action. On mobile it becomes a menu drawer. Footer navigation should repeat the essential legal, support, documentation, and security links.

### Auth shell

The auth shell is deliberately quiet: centered form panel, brand mark, concise help/status links, and no full marketing navigation. It should preserve the same typography and color tokens while prioritizing completion, validation, and recovery states.

### Platform shell

The platform shell uses the design specification's utility rail: 64px collapsed and approximately 260px expanded. It contains Overview, Deployments, Services, Models, Analytics, Toolkit, Docs, and Settings. The main workspace is independently scrollable. The top bar should expose workspace context, environment/status, global search or command palette, notifications, and the user menu.

The rail must be keyboard navigable and have an accessible expanded/collapsed label. On small screens, it becomes a drawer or bottom-level navigation rather than remaining permanently fixed.

## 6. Design-System Implementation

Promote the existing specification into typed tokens so the visual language is reusable rather than copied between pages.

| Token group | Baseline |
|---|---|
| Canvas | `#0B1020` deep void with subtle blue radial glow |
| Primary | `#2563EB` electric blue |
| Secondary | `#8B5CF6` luminous violet |
| Positive/live | `#10B981` green-cyan |
| Text | `#F1F5F9` primary, slate variants for secondary text |
| Fonts | Inter for UI/body, Montserrat for display, JetBrains Mono for code/telemetry |
| Grid | 12 columns desktop, 8 tablet, 4 mobile; max width 1440px |
| Spacing | 4px/8px rhythm; 8, 16, 24, and 40px core steps |
| Radius | 8px controls, 16px cards, 24px elevated panels, pill status badges |
| Surfaces | Layered translucent slate with subtle borders and restrained glow |

Core components should include `Button`, `IconButton`, `Badge`, `Input`, `Select`, `Dialog`, `Card`, `MetricCard`, `DataTable`, `StatusPill`, `TerminalBlock`, `EmptyState`, `Toast`, `Skeleton`, `ErrorState`, and `CommandPalette`. Every component needs keyboard, focus, loading, empty, error, and reduced-motion behavior where applicable.

Avoid relying on runtime Tailwind CDN scripts in production. The current mockups use CDN Tailwind and Google Fonts; the production build should bundle or self-host the chosen fonts and compile styles so deployments are deterministic and Content Security Policy can be tightened.

## 7. Domain and Data Boundaries

The initial domain model should remain small and explicit:

```text
User
  id, email, displayName, role, status

Workspace
  id, name, plan, environment, createdAt

Membership
  userId, workspaceId, role, permissions

Service
  id, workspaceId, name, environment, status, version, latency

Deployment
  id, workspaceId, serviceId, version, status, actorId, startedAt, completedAt

Model
  id, workspaceId, name, version, status, provider, updatedAt

MetricSeries
  workspaceId, subjectType, subjectId, metric, timestamp, value

Incident
  id, status, severity, title, startedAt, resolvedAt
```

Use workspace-scoped authorization on every authenticated read and write. The UI should never infer authorization from hidden controls alone; the API must enforce it. Audit events should be recorded for deployment actions, API-key changes, role changes, and other security-sensitive operations.

## 8. API and Integration Boundary

Expose a versioned server boundary such as `/api/v1`. Suggested resource groups are `/auth`, `/workspaces`, `/services`, `/deployments`, `/models`, `/metrics`, `/incidents`, and `/audit-events`.

For the first milestone, use a local mock adapter with deterministic fixtures. The adapter should implement the same interfaces as the eventual API, allowing real services to replace fixtures without rewriting page components. Long-running deployment or analytics updates should use polling or server-sent events only after the basic request/response flows are stable.

External credentials must stay in environment-managed secrets. Never place API keys, access tokens, or private service URLs in this public repository or in client-side bundles.

## 9. Security and Privacy Baseline

The public repository already has secret-scanning and push protection enabled. Keep those controls enabled and add dependency security updates before production code is introduced. The target application should also implement:

- Secure, HTTP-only, same-site session cookies or a vetted identity provider.
- Server-side authorization checks for workspace and role permissions.
- CSRF protection for cookie-authenticated mutations.
- Rate limiting on sign-in, account creation, password recovery, and contact endpoints.
- Input validation at the API boundary and output encoding for user-controlled content.
- Content Security Policy, strict transport security, and secure cookie flags.
- Redacted logs and no secrets or sensitive customer data in telemetry payloads.
- Audit history for access, deployments, API keys, and permission changes.
- Clear retention and deletion policies for contact, account, and telemetry data.

## 10. Accessibility and Performance

Target WCAG 2.2 AA for the public and platform surfaces. Ensure visible focus states, semantic headings, correct form labels, keyboard access to the rail and command palette, sufficient contrast, reduced-motion support, and meaningful status announcements for async actions.

Use static generation or server rendering for public pages, lazy-load the dashboard and heavy visualizations, reserve space for charts to avoid layout shift, optimize reference imagery, and avoid loading all platform modules on the landing page. Measure Core Web Vitals on the public routes and establish a performance budget before adding animation or large visualization libraries.

## 11. Delivery Plan

### Phase 0 — Foundation

Create the application scaffold, TypeScript configuration, route shells, token layer, linting, formatting, unit-test setup, and a minimal CI workflow. Move only approved logos and optimized assets into `public/brand`.

### Phase 1 — Public MVP

Implement `/`, `/solutions`, `/platform`, `/security`, `/documentation`, `/status`, and `/contact` using the existing landing page and brand references. Replace placeholder `href="#"` links with real routes and make all CTA states explicit.

### Phase 2 — Identity

Implement sign-in, account creation, password recovery, verification, legal pages, validation, loading states, and error recovery. Connect to the chosen identity provider only after the screens work against a mock adapter.

### Phase 3 — Platform MVP

Implement the platform shell, overview, deployments, services, models, and settings routes. Start with fixture data, then connect read-only API endpoints, then enable controlled mutations such as deployment actions.

### Phase 4 — Hardening

Add end-to-end tests, accessibility checks, error monitoring, audit logging, rate limits, CSP, dependency updates, performance budgets, and production deployment previews.

## 12. GitHub and Deployment Recommendations

The repository currently has one branch and no Actions workflow. Before production implementation:

1. Add a pull-request workflow that runs formatting, linting, type checking, unit tests, and a production build.
2. Enable Dependabot security updates.
3. Protect `main` with required pull-request review and required CI checks.
4. Enable automatic branch deletion after merge.
5. Add an environment-based deployment target with preview deployments for pull requests and production deployment from `main`.
6. Keep the public design archive under `stitch_brand_website_and_screen_design/`, but place production code under `src/` so references and application code are not confused.
7. Add a concise README with local setup, environment variables, test commands, and deployment instructions.

A suitable initial deployment model is a static/server-rendered public site plus a protected application/API deployment. If the platform API is not ready, deploy the public surface first and keep `/app/*` behind an explicit “access coming soon” or authentication gate rather than exposing mock operational data as real telemetry.

## 13. Definition of Done for the First Production Release

The first release is ready when the public navigation has no dead links, the landing page is responsive, brand assets are optimized and licensed, forms have real validation and error states, all protected routes enforce authentication and workspace access, CI passes on every pull request, secrets are externalized, public pages meet the agreed accessibility and performance targets, and the deployment process can be repeated from a clean checkout.

## 14. Immediate Next Step

Create the application scaffold and design-token layer first. Then implement the public home page from `marvelcode_turning_ideas_into_intelligent_solutions/code.html` as the reference screen, while treating the platform portal, sign-in, and account-creation mockups as separate route references rather than trying to stitch all HTML files into one page.
