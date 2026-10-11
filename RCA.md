# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output
* `dpkg` prompts interactively asking which copy of `/etc/nginx/nginx.conf` to keep.
* The build log ends with `end of file on stdin at conffile prompt`.
* `DEBIAN_FRONTEND=noninteractive` does not answer a dpkg conffile prompt, so that setting alone does not fix the build.

### How I diagnosed it
* Executed `./scripts/run-tests.sh` and inspected Docker build logs.
* Inspected `proxy/Dockerfile`: observed `COPY nginx.conf /etc/nginx/nginx.conf` ran prior to `apt-get install -y nginx`.

### Root cause
* Copying custom configuration files before installing the `nginx` package causes `dpkg` to treat the file as a pre-existing locally modified conffile, triggering interactive prompt collisions during package extraction.

### Fix
* Reordered instructions to install `nginx` prior to copying custom configs.
* Later replaced that Ubuntu file with `nginx:1.27-alpine-slim` (bonus question). The final Dockerfile has no `apt-get`.

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
* Integration tests failed on `/api/info` with `HTTP Error 502: Bad Gateway`.
* Proxy connection failed with upstream connection refused.

### How I diagnosed it
* Inspected `proxy/nginx.conf`: upstream was hardcoded to `server 127.0.0.1:8080;`.
* Compose puts the services on a user-defined network for this project. That network has DNS. Docker's default bridge does not.

### Root cause
* `127.0.0.1` inside the proxy container is the proxy itself, so nginx was connecting to its own port 8080.

### Fix
* Updated `upstream app_backend` in `proxy/nginx.conf` from `127.0.0.1:8080` to `backend:8080`. The service name resolves because of the Compose network DNS.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
* Race condition leading to intermittent or immediate `502 Bad Gateway` / `Connection refused` on `/api/info`.

### How I diagnosed it
* Inspected container startup sequence: `test-runner` executed requests immediately while `backend` was still loading configs in a delayed loop (`[Backend] loaded config key...`).
* Inspected `backend/app.py`: artificial startup delay governed by `CONFIG_FETCH_SECONDS`.

### Root cause
* Race condition: Compose `depends_on` only verifies container startup, not application readiness. The backend took ~8 seconds before binding port 8080.

### Fix, and why I chose it over the alternatives
* Left `CONFIG_FETCH_SECONDS` unset, so the 2-second default stays. `backend/app.py` was not edited.
* Backend healthcheck runs `python -c` with `urllib` against `http://127.0.0.1:8080/health`. The Python image has no curl. `interval: 1s`, `retries: 10`, `start_period: 30s`.
* Failures during `start_period` do not count toward `retries`. The first success marks the backend healthy, so the run continues at about 8 seconds instead of waiting out the whole 30.
* The proxy uses `depends_on` with `condition: service_healthy`, so it does not start until that check passes. The test-runner still waits only for the proxy container to start.
* **Why this over alternatives:** A sleep in the test script would not weaken the assertions, but a fixed sleep is brittle. Setting `CONFIG_FETCH_SECONDS=0` would skip the delay instead of waiting for the port.

---

## Defect 4: CI pipeline fails

### Symptom / error output
* Local execution failed with `zsh: permission denied: ./scripts/run-tests.sh`.
* GitHub Actions runner failed immediately on the test execution step.

### How I diagnosed it
* Ran `git ls-files -s scripts/run-tests.sh` and checked POSIX file modes (`100644` without executable bit).

### Root cause
* The test runner script `scripts/run-tests.sh` was committed to the Git index without executable permissions (`+x`).

### Fix
* Updated Git tree metadata: `git update-index --chmod=+x scripts/run-tests.sh` (mode `100755`). A clone gets that mode.
* `.github/workflows/ci.yml` runs `test -x ./scripts/run-tests.sh` before the suite. It does not `chmod` the file, so a missing execute bit fails CI.

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

> 1. **Proxy Image Size Reduction:**
>    * Replaced `ubuntu:24.04` with `nginx:1.27-alpine-slim`.
>    * `docker images` SIZE for the Ubuntu proxy, built from the Dockerfile before that change: `proxy-ubuntu-before 150MB`. `docker image inspect` reported `Size=150295863`. That image was Ubuntu with only `nginx` installed. `curl`, `htop`, `vim`, and `git` were already removed from that Dockerfile.
>    * `docker images` SIZE for the image the suite uses now: `f5-devops-challenge-proxy 20.2MB`. `docker image inspect` reported `Size=20171171`.
>
> 2. **Running as Non-Root User:**
>    * `USER nginx` in `proxy/Dockerfile`. Nginx still `listen`s on 80.
>    * `cap_add: NET_BIND_SERVICE` is listed on the proxy in `docker-compose.yml`. That capability is already in Docker's default set, so the line does not grant a new one.
>    * Host publish is `8081:80`. Tests still use `PROXY_URL=http://proxy:80` inside the Compose network.
>    * `pid /tmp/nginx.pid;` so the nginx user can write the pid file.
>    * `chown -R nginx:nginx /var/cache/nginx /var/log/nginx` before the `USER` line.
>    * `docker compose run --rm proxy whoami` printed `nginx`. That shows the user. It does not by itself show that port 80 opened.
>
> 3. **Additional Security & Performance Improvements:**
>    * `server_tokens off;` so the `Server` header does not include the nginx version.

---

## Anything you would do differently with more time
* Add Docker layer caching in `.github/workflows/ci.yml` via GitHub Actions cache to speed up remote builds.
* Add static analysis tooling (Hadolint for Dockerfile, ShellCheck for shell scripts) as pre-test stages.