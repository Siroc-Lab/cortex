# SUT Discovery

The SUT (System Under Test) is a running web application — either a local dev server or a remote staging/production URL. The skill must identify it before investigation begins.

## Discovery Sources

Check these in order:

1. **`$ARGUMENTS`** — If the operator passed a URL, use it directly. Still confirm it's reachable.

2. **CLAUDE.md** — Look for:
   - Dev server commands (`npm run dev`, `yarn dev`, `pnpm dev`)
   - Port numbers (`localhost:3000`, `localhost:5173`)
   - Staging/preview URLs
   - Environment setup instructions

3. **package.json scripts** — Look for:
   - `dev`, `start`, `serve`, `start:local` scripts
   - Port configurations in the script commands

4. **docker-compose.yml** — Look for:
   - Service port mappings
   - Frontend service definitions

5. **docs/** — Look for:
   - Setup guides mentioning URLs
   - Architecture docs with service maps

## Inference Patterns

Common patterns to recognize:

| Framework | Default Port | Start Command |
|-----------|-------------|---------------|
| Vite | 5173 | `vite dev` |
| Next.js | 3000 | `next dev` |
| Create React App | 3000 | `react-scripts start` |
| Strapi | 1337 | `strapi develop` |
| Express | 3000 | `node server.js` |

## Confirmation Flow

**Never assume.** After gathering information, present findings:

> "I found `npm run dev` in package.json which starts a Vite server on port 5173. Is this the app to test, or should I use a different URL?"

If no SUT information is found:

> "I couldn't determine where the app is running. Can you provide the URL or tell me how to start the dev server?"

If the operator provides a URL, verify it's reachable. If it's reachable, proceed. If not, attempt server startup below before blocking.

## Server Startup

When the SUT URL is known but the port is not responding, attempt to start the server rather than immediately blocking:

1. **Identify the start command** — already gathered from CLAUDE.md, package.json, or docker-compose in the discovery steps above
2. **Start in background** — run the command via Bash with `run_in_background: true`; do not wait for it interactively
3. **Wait for the port** — poll until ready (30s max):
   ```bash
   for i in $(seq 1 30); do curl -s -o /dev/null http://localhost:<port> && break || sleep 1; done
   ```
4. **Open a browser page** — once the port responds, use `new_page` + `navigate_page` (see `references/tooling.md`)
5. **Confirm with operator**:
   > "I started the dev server with `npm run dev` and opened the app at localhost:5173. Proceeding with QA."

If the start command cannot be determined, or the port never responds within 30 seconds, fall through to blocking below.

## SUT is Blocking

Only block if no start command could be found **and** the operator cannot provide one, or if the startup attempt failed.

When blocking:
- Report what was attempted (command tried, port checked, timeout reached)
- Ask the operator for help

Do NOT proceed with source-code-only analysis and call it "investigation."
