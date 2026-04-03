# Docker Image: Build, Customize, and Push to ghcr.io

AlphaDiana uses a patched Docker image (`openclaw-reasoning:patched`) as the ROCK sandbox environment for OpenClaw agents. This guide explains what the patched image changes, how to build it, how to customize it for your needs, and how to push it to the GitHub Container Registry (ghcr.io).

---

## What the Patched Image Changes

The patched image (`openclaw_deploy/Dockerfile.patched`) is based on the prebuilt OpenClaw image and applies two fixes on top of it:

### Fix 1: Python Sandbox Environment

The base OpenClaw image ships with `python3` but is missing several things agents need when executing code inside the sandbox:

| Issue | Fix |
|-------|-----|
| No `python` command (only `python3`) | Symlink `python` → `python3` |
| No `pip` (base image is Debian-slim) | Install `python3-pip` via apt |
| No math packages | Install `sympy` and `numpy` via pip |

Without this fix, any agent that runs `python ...` or `import sympy` inside the sandbox will fail.

### Fix 2: OpenAI SDK Stream Timeout (30 min)

**Problem:** Node.js v24's native `fetch` uses an internal undici HTTP client with a hard-coded `bodyTimeout` of 300 seconds. OpenClaw's `ensureGlobalUndiciStreamTimeouts` patches the *npm* undici package but does **not** patch `globalThis.fetch`'s internal undici. When a streaming LLM response takes longer than 5 minutes (common for complex reasoning tasks), the stream is silently truncated.

**Fix:** Patch the OpenAI SDK constructor in `@mariozechner/pi-ai` to add `timeout: 1800000` (30 minutes). This makes the SDK manage its own timeout via `AbortController` + `setTimeout`, bypassing the undici `bodyTimeout` entirely.

Patched files inside the container:
- `/app/node_modules/@mariozechner/pi-ai/dist/providers/openai-completions.js`
- `/app/node_modules/openclaw/node_modules/@mariozechner/pi-ai/dist/providers/openai-completions.js`

---

## Building the Patched Image

From the repo root:

```bash
docker build -t openclaw-reasoning:patched -f openclaw_deploy/Dockerfile.patched .
```

This takes the prebuilt base image and applies the two fixes above. Build time is typically under 2 minutes.

Then reference `openclaw-reasoning:patched` in your config's `rock_image` field instead of the remote base image.

---

## Customizing the Image

To add your own packages or patches, create a new Dockerfile that extends the patched image. For example, to add `scipy` and `matplotlib`:

```dockerfile
FROM openclaw-reasoning:patched

USER root
RUN pip install --no-cache-dir --break-system-packages scipy matplotlib
USER node
```

Build it:

```bash
docker build -t openclaw-reasoning:custom -f Dockerfile.custom .
```

Then update your experiment config to use the custom image:

```yaml
agent:
  name: openclaw
  config:
    rock_image: "openclaw-reasoning:custom"
    # ... rest of config
```

**Tips:**

- Always switch to `USER root` before running `apt-get` or `pip install`, and switch back to `USER node` afterward — the OpenClaw process runs as the `node` user.
- Use `--break-system-packages` with pip because the base image uses a system Python (no virtualenv).
- If you need to patch OpenClaw's Node.js code, follow the pattern in `Dockerfile.patched` — use `node -e` with `fs.readFileSync` / `fs.writeFileSync` to find-and-replace.

---

## Pushing to GitHub Container Registry (ghcr.io)

### Prerequisites

- Docker installed locally
- A GitHub account with write access to the target repository/organization
- A GitHub Personal Access Token (PAT) with the `write:packages` scope

### Creating a PAT

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Click **Generate new token**
3. Select the scope: `write:packages` (this also requires `read:packages` and `repo` for private repos)
4. Copy and save the token — you won't see it again

### Step 1: Log in to ghcr.io

```bash
echo "<YOUR_PAT>" | docker login ghcr.io -u <YOUR_GITHUB_USERNAME> --password-stdin
```

Example:

```bash
echo "ghp_xxxxxxxxxxxxxxxxxxxx" | docker login ghcr.io -u tsrigo --password-stdin
```

### Step 2: Tag the image

Tag the local image with the full ghcr.io path. Use the date as the tag in `YYYYMMDD` format:

```bash
docker tag openclaw-reasoning:patched ghcr.io/<OWNER>/<IMAGE_NAME>:<TAG>
```

Example:

```bash
docker tag openclaw-reasoning:patched ghcr.io/tsrigo/openclaw-reasoning:20260320
```

The `<OWNER>` is your GitHub username or organization name. The image will appear under that account's packages.

### Step 3: Push the image

```bash
docker push ghcr.io/<OWNER>/<IMAGE_NAME>:<TAG>
```

Example:

```bash
docker push ghcr.io/tsrigo/openclaw-reasoning:20260320
```

### Step 4: (Optional) Make the package public

By default, a newly pushed package inherits the visibility of the repository. To make it publicly pullable without authentication:

1. Go to **github.com/\<OWNER\>** → **Packages** → find `openclaw-reasoning`
2. Click **Package settings**
3. Under **Danger Zone**, click **Change visibility** → **Public**

Once public, anyone can pull it with:

```bash
docker pull ghcr.io/tsrigo/openclaw-reasoning:20260320
```

---

## Quick Reference

```bash
# Build
docker build -t openclaw-reasoning:patched -f openclaw_deploy/Dockerfile.patched .

# Login
echo "$GITHUB_TOKEN" | docker login ghcr.io -u $GITHUB_USERNAME --password-stdin

# Tag
docker tag openclaw-reasoning:patched ghcr.io/$GITHUB_USERNAME/openclaw-reasoning:$(date +%Y%m%d)

# Push
docker push ghcr.io/$GITHUB_USERNAME/openclaw-reasoning:$(date +%Y%m%d)
```

---

## Notes

- The current production image is `ghcr.io/tsrigo/openclaw-reasoning:20260320`. If you build and push a newer version, update the image reference in `README.md` and `openclaw_deploy/rock_agent_config.prebuilt.yaml` accordingly.
- The `Dockerfile.patched` source is at `openclaw_deploy/Dockerfile.patched` — review it before building to understand exactly what gets changed.
