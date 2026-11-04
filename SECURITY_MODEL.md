# Threat model: Platform-Adventure repositories

_Draft by Devin from the code; also stored on the "Platform-Adventure threat model" security scan profile. Items marked **(assumed)** have not been confirmed by the owner yet._

## System overview and trust boundaries

Nine repositories, which make up five unrelated systems with no shared backend, no shared data store and no secrets passed between them:

| Component | Code | What it is |
|---|---|---|
| **Express Web App** | `express/src/index.ts`, `express-js-on-vercel/src/{index,server,speed-insights}.ts`, `components/about.htm`, `public/` | Vercel "Express" example. Serves a static HTML home page, `/about` (`res.sendFile` of a fixed path), `/api-data` (hard-coded JSON) and `/healthz`. It has no auth, no database, no user input and no env secrets (only `PORT`). `express-js-on-vercel` also injects the Vercel Speed Insights script. |
| **Clicker Game** | `friendly-octo-disco/src/**`, `index.html` (bug-free-fishstick holds only a README for the same game) | Vite single-page app that runs entirely in the browser. Game state is stored in `localStorage` (`engine.js` `saveGameState`/`loadGameState`) and the UI is rendered with `innerHTML` templates (`ui/appScreen.js`). There is no server, no money and no network calls. |
| **React Router Library** | `react-router/packages/*` (core `react-router`, `react-router-dev` Vite plugin/CLI, server adapters `-node`, `-express`, `-cloudflare`, `-architect`, `-serve`, `-fs-routes`, `create-react-router`) | Fork of the upstream `remix-run/react-router` library. The code is meant to run inside other people's apps: SSR request handling, route matching, data loaders/actions, redirects, and RSC. |
| **WPScan CLI** | `wpscan/app/**`, `wpscan/lib/**`, `bin/`, `Dockerfile` | Fork of the WordPress security scanner. It is a Ruby CLI run by an operator against a target site. It parses untrusted HTTP responses from that site, downloads/updates a local vulnerability DB (`lib/wpscan/db/updater.rb`), uses a `Marshal` file cache (`lib/wpscan/cache/file_store.rb`), and can use a WPScan API token. |
| **CI / Supply Chain** | `react-router/.github/workflows/*`, `wpscan/.github/workflows/{build,docker,gempush}.yml`, `wpscan/Dockerfile`, lockfiles | GitHub Actions workflows. Release secrets include `RUBYGEMS_AUTH_TOKEN` and the npm/`FORMAT_PAT` tokens. The workflows also publish Docker images. |

Repos with no code to scan: `fantastic-potato-` and `new-acode-repo` (README only), and `oliviamazzotta47` (profile repo with a template `SECURITY.md`).

Trust boundaries:
- Internet → Express Web App (public, anonymous).
- Browser/localStorage → Clicker Game. The player controls all state.
- App developer / end-user request → React Router Library. A request reaching an app built on the library is untrusted.
- Target WordPress site → WPScan CLI. Responses are attacker-controlled when the target is malicious.
- Pull-request author → CI / Supply Chain. Forks and PRs run in workflows.

## Entry points and who can reach them
- Express: `GET /`, `/about`, `/api-data`, `/healthz`, and static files from `public/`. Anyone on the internet can reach them when deployed **(assumed: deployed publicly on Vercel)**.
- Clicker Game: the page itself, plus the game's save key in `localStorage`. Only the local player can reach these.
- React Router: the exports of the published packages, the request handlers in the server adapters, and the `react-router-dev` CLI/Vite plugin.
- WPScan: CLI options (operator), HTTP responses from the target, DB update downloads, and the cache directory.
- CI: `pull_request`/`pull_request_target` triggers, release triggers, and workflow inputs.

## Assets and data sensitivity
- Release credentials in GitHub secrets (RubyGems, npm, `FORMAT_PAT`): **high**. A leak allows publishing malicious packages.
- Integrity of published artifacts (gem, Docker image, npm packages), **if** these forks are published **(open question)**.
- WPScan operator machine and API token: medium.
- Express app availability and content: low. The app holds no data.
- Clicker Game state: none. The game is single-player with no real value.

## Attacker profiles
1. Anonymous internet user hitting the Express app.
2. Malicious WordPress site / MITM attacking a WPScan operator.
3. Malicious PR author or compromised dependency targeting CI and releases.
4. Remote attacker sending crafted requests to an app built on the React Router fork. This only matters if the fork is used or published.

## Existing controls
- Express apps hold no secrets or user input, and `/about` serves a fixed file path.
- The WPScan Docker image runs as the non-root user `wpscan`, and its GitHub Actions are pinned by SHA.
- react-router comes from upstream with its full test suite and upstream security process (`SECURITY.md`).
- Dependabot is active on react-router.

## Out of scope / accepted risk
- _None confirmed by the owner yet._

## Open questions
- Are the react-router and wpscan forks published or used in production, or kept only as reference copies?
- Are the Express apps publicly deployed?
- Is client-side cheating in the Clicker Game an accepted risk?
- Is there any infrastructure repository? None was found, so this model was written without infrastructure-as-code.
