# Step 0: Prerequisites

This Labspace uses a host-backed terminal so `sbx` can create Docker
Sandboxes from your machine. A Docker Sandbox runs the agent in an
isolated microVM with its own Docker daemon, filesystem, and network.

## Before you begin
 
In this section, you will complete the following setup:
 
- Confirm Docker and the `sbx` CLI are installed and authenticated
- Create a Docker Personal Access Token (PAT) that can pull Docker Hardened Images
- Register host-side SBX secrets so the real PAT never enters a sandbox VM
- Validate the two kits packaged with this lab and reset the sample app to a clean state


## Confirm Docker and SBX

Confirm Docker and SBX are available:

```bash
docker version
sbx version
```

Check SBX authentication:

```bash
sbx diagnose
```

If authentication fails, sign in:

```bash
sbx login
```

## Prepare Docker credentials


Prepare Docker credentials for the DHI step:
 
1. Create a Docker Personal Access Token (PAT) from Docker Home:
   **Account settings** -> **Personal access tokens** -> **Generate token**.
2. Use a Docker account or organization that can pull Docker Hardened
   Images.
3. Give the token permission to push to your Docker Hub namespace. The
   lab publishes two comparison tags there.
4. Copy the token once. Docker will not show it again.

> [!IMPORTANT]
> Use the PAT as the Docker registry password. Do not use the Docker
> account password for this lab.

Set the Docker username for the rest of this lab:

::variableDefinition[dockerUsername]{prompt="Docker username"}

The lab uses this image repository for shareable Hub / Scout evidence:

```text
docker.io/$$dockerUsername$$/todo-demo-application
```

> [!NOTE]
> If Docker Hub rejects the first push with `denied`, create that
> repository in Docker Hub under the same namespace and rerun the push
> block.

## Register the registry secrets

Register the Docker Hub and `dhi.io` auth placeholders. This block reads
the PAT interactively, which the run button cannot drive (it would feed
the script's own lines into the hidden prompt), so the block below has no
run button. **Use its Copy button and paste it into the terminal
yourself.** When prompted, paste the PAT and press Enter. Input is hidden.

> [!IMPORTANT]
> This does not put the PAT in the sandbox VM. The `dhi` kit writes a
> Docker config with fake auth placeholders; host-side SBX custom secrets
> replace those placeholders only when outbound registry requests pass
> through the SBX proxy.

```bash no-run-button
DOCKERHUB_USERNAME="$$dockerUsername$$" bash <<'SCRIPT'
set -euo pipefail

DOCKER_HUB_AUTH_PLACEHOLDER_B64="c2J4X2RvY2tlcl9odWJfdXNlcjpzYnhfZG9ja2VyX2h1Yl9wYXQK"
DHI_AUTH_PLACEHOLDER_B64="c2J4X2RoaV91c2VyOnNieF9kaGlfcGF0Cg=="

if [ -z "${DOCKERHUB_USERNAME:-}" ] || [ "$DOCKERHUB_USERNAME" = "dockerUsername" ]; then
  echo "Set the Docker username field in the lab instructions first." >&2
  exit 1
fi

printf 'Paste Docker PAT for %s: ' "$DOCKERHUB_USERNAME" > /dev/tty
IFS= read -r -s DOCKERHUB_PAT < /dev/tty
printf '\n' > /dev/tty

if [ -z "$DOCKERHUB_PAT" ]; then
  echo "No PAT entered; nothing was changed." >&2
  exit 1
fi

REGISTRY_AUTH_B64="$(printf '%s:%s' "$DOCKERHUB_USERNAME" "$DOCKERHUB_PAT" | base64 | tr -d '\n')"

sbx secret set-custom -g \
  --host auth.docker.io \
  --env DOCKER_HUB_CREDS \
  --placeholder "$DOCKER_HUB_AUTH_PLACEHOLDER_B64" \
  --value "$REGISTRY_AUTH_B64"

sbx secret set-custom -g \
  --host dhi.io \
  --env DHI_CREDS \
  --placeholder "$DHI_AUTH_PLACEHOLDER_B64" \
  --value "$REGISTRY_AUTH_B64"

unset DOCKERHUB_PAT REGISTRY_AUTH_B64
SCRIPT
```

Global secrets are used for new sandboxes. If you already created
`p4-dhi`, remove and recreate it after setting or changing secrets:

```bash
sbx rm -f p4-dhi
```

> [!IMPORTANT]
> Do not paste the Docker PAT into the sandbox. The sandbox should only
> receive proxy-managed placeholders; the real token stays on the host and
> the proxy rewrites the outbound registry auth request.

## Validate the kits

Validate the two kits packaged with the lab:

```bash
cd ~/.labspace/project
sbx kit validate ./kits/container-best-practices
sbx kit validate ./kits/dhi
```

## Reset the sample app

Reset the sample app before each demo pass. This removes the old demo
sandboxes and the agent-generated `Dockerfile` from any previous run, so
the agent can create `Dockerfile` again from a clean app that ships no
Dockerfile of its own. Only the agent's artifacts are removed; the app
sources are left intact, so you can repeat the demo without redoing this
Step 0:

```bash
cd ~/.labspace/project && ./scripts/reset-demo.sh
```

> [!TIP]
> Optional: pre-pull the Claude sandbox template so the first demo command
> starts faster. Run the block below.


```bash
sbx create --name prewarm claude /tmp && sbx rm -f prewarm
```

> [!WARNING]
> If this fails with a mount policy error such as:
>
> ```text
> ERROR: create runtime: create runtime: sandboxd error: status 403:
> mount policy denied: /private/tmp: no applicable policies for
> op(action=fs:mount:write, resource=fs:path:/private/tmp)
> ```
>
> you likely have **AI Governance** configured (for example, an
> organization policy), and that policy does not grant the sandbox
> permission to mount the requested path. Either adjust the governance
> policy to allow the path, or skip this optional prewarm step. Check your
> active policies with `sbx policy ls` and review your governance
> configuration with your administrator.


## If sandboxes can't reach Docker Hub (AI Governance)

This lab needs the sandbox to pull base images and tools from Docker Hub,
GitHub, npm, and `dhi.io`. If your Docker organization enforces **AI
Governance**, a restrictive network policy can block these, and the lab will
fail partway through — the agent authenticates fine but the build cannot pull
its base image.

### Symptoms

The agent reports a network-policy denial during `docker build`, for example:

```text
Blocked by network policy: domain registry-1.docker.io:443
  detail: no matching allow rule — blocked by default deny policy
```

### Confirm it is a governance policy

```bash
sbx policy ls
```

If the output shows `Governance — Managed by <org>` and the active rules only
allow AI-service endpoints, your org policy is the cause. Local
`sbx policy allow network ...` rules will **not** help — governance policy
takes precedence and does not delegate network rules to local policy:

```bash
sbx policy ls --include-inactive
# look for: "inactive — corporate policy takes precedence and does not
# delegate this rule type to local policy."
```

### Fix: allow the lab's domains in the AI Governance portal

An organization owner must add an **allow** network policy. In the Docker
Admin Console: select your organization, open **AI Governance** (sandbox
policies), and create a new network **Allow** rule (for example "Allow SBX
Kits DHI Lab") with these resources:

```text
# Docker Hub: registry, token auth, blob/CDN storage
docker.io:443
*.docker.io:443
registry-1.docker.io:443
auth.docker.io:443
docker.com:443
*.docker.com:443
hub.docker.com:443
api.docker.com:443
production.cloudflare.docker.com:443
production.cloudfront.docker.com:443

# GitHub: hadolint and dhictl release downloads
github.com:443
api.github.com:443
*.githubusercontent.com:443

# npm registry (app dependency install)
registry.npmjs.org:443

# DHI step
dhi.io:443
```

After the policy syncs, confirm from the host and recreate the sandbox:

```bash
sbx policy ls          # the new allow rule should appear as remote/active
sbx rm -f p1-yolo      # recreate so it starts under the updated policy
```

> [!NOTE]
> No org access to change governance? Run the lab while signed in to a Docker
> account that is **not** subject to AI Governance (a personal account, or an
> org without sandbox policies). The built-in local defaults already permit
> Docker Hub, npm, and GitHub, so the lab runs without extra rules.

