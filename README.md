# Tasks app: two small services

A to-do API that calls a tiny stateless function to clean up task titles.

| Service | Folder | Port | What it does |
|---|---|---|---|
| tasks-api | `services/tasks-api` | 5000 | CRUD for tasks. Calls `title-normalizer` when a task is created. |
| title-normalizer | `services/title-normalizer` | 5001 | One function: `POST /invoke` trims a title, collapses spaces, capitalizes the first letter. |

If `title-normalizer` is down, `tasks-api` still creates the task with the raw title and `"normalized": false`.

## Endpoints

tasks-api:

- `GET /health`
- `GET /tasks`
- `POST /tasks` with `{"title": "..."}`: 201, or 400 for an empty title or a non-JSON body
- `GET /tasks/<id>`: 200 or 404
- `DELETE /tasks/<id>`: 204 or 404

title-normalizer:

- `GET /health`
- `POST /invoke` with `{"title": "  buy   milk "}` returns `{"title": "Buy milk"}`; 400 if `title` is not a string

## Run (two terminals)

Each service has its own `requirements.txt`.

```bash
# terminal 1
cd services/title-normalizer
pip install -r requirements.txt
flask --app app:create_app run --port 5001

# terminal 2
cd services/tasks-api
pip install -r requirements.txt
NORMALIZER_URL=http://localhost:5001 flask --app app:create_app run --port 5000
```

Firewall commands
```bash
sudo iptables-legacy -P FORWARD ACCEPT
sudo iptables-legacy -S FORWARD | head -1
```

Try it:

```bash
curl -X POST localhost:5000/tasks -H 'Content-Type: application/json' -d '{"title": "  buy   milk "}'
curl localhost:5000/tasks
```

Stop terminal 1 and create another task: it still works, with `"normalized": false`.

## Test

Run in each service folder:

```bash
pytest
```

## What each meeting covers

The repo grows meeting by meeting. First a pipeline (`.github/workflows/ci.yml`), later containers, infrastructure and monitoring. Find your meeting below to see what it covers. Nothing here is homework, and later topics are planned: order and details may change.

### Meeting 1: first CI

A workflow that runs on every push: check out the code, set up Python, install the requirements, run `pytest`.

Tasks: break a test on a branch and read the red run. Find which step failed and why.

### Meeting 2: CI in depth

The same workflow for both services and several Python versions (a matrix). The test output is saved as an artifact. A Bandit security scan writes an HTML report, also saved as an artifact. `continue-on-error` lets the job go on when Bandit finds something.

Tasks: read the matrix and `upload-artifact` parts of the [workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax). Download an artifact from a run page and open it.

### Meeting 3: CD with GitHub Pages

A `deploy` job that waits for `run_tests` (`needs`), downloads the two Bandit reports, builds an `index.html` and publishes everything with GitHub Pages. A `generate_api_docs` job makes API docs for both services with pdoc, and the index page links to them. The deploy runs only when a pull request is merged: the `pull_request` trigger has the activity types `opened`, `synchronize` and `closed`, and `deploy` checks the `merged` field. Pages needs a public repo on a free account, and what you publish is public.

Two settings are not in the file. Set them in your own repo:

- Settings > Pages > Source: GitHub Actions (the repo must be public).
- Settings > Environments > github-pages > Deployment branches and tags: add the rule `refs/pull/*/merge`. Without it the `deploy` job of a merged pull request is refused with: Branch "refs/pull/N/merge" is not allowed to deploy to github-pages due to environment protection rules.

If you missed this meeting: replace `.github/workflows/ci.yml` with the one in this repo, delete `.github/workflows/ci_livecode.yml` if you have it, and set the two settings above. None of this is homework.

Docs: [Using custom workflows with GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages), [events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).

Tasks:

1. Auto-generated API docs: add any tool that documents Flask code, run a command that creates the HTML, upload it, publish it on your site and link it from the index page. Do not change `services/`. [pdoc](https://pdoc.dev/) is one easy option.
2. Deploy only when a pull request is merged (look at the `pull_request` activity types and the `merged` field).
3. Read your whole `ci.yml` and explain each job to yourself.

### Meeting 4: Docker images and containers

A Dockerfile describes an image: a base image plus the steps that add your service to it. You build the image, run it as a container, and reach it on a port.

If you missed this meeting: copy these four files from this repo into your own, each as a whole file: `services/tasks-api/Dockerfile`, `services/tasks-api/.dockerignore`, `services/title-normalizer/Dockerfile`, `services/title-normalizer/.dockerignore`. There is no setting to change and `ci.yml` does not change. None of this is homework.

Docs: [Dockerfile reference](https://docs.docker.com/reference/dockerfile/), [Building best practices](https://docs.docker.com/build/building/best-practices/), [Forwarding ports in your codespace](https://docs.github.com/en/codespaces/developing-in-a-codespace/forwarding-ports-in-your-codespace).

Tasks:

1. A Dockerfile for `tasks-api`: build the image, run it, then `curl` `/health` and `POST /tasks`. Do not change `services/`.
2. The same for `title-normalizer`.
3. A `.dockerignore` for each service.
4. Run a container as a user that is not root.

### Meeting 5: Docker Compose and publishing images

A `compose.yaml` describes several services and starts them with one command. Each service name is a host name on the network Compose creates. A `build_images` job builds both images on every run, and a job can publish an image to a registry (`ghcr.io`) with the token the run has.

In a Codespace, traffic between containers on a Compose network is dropped by default. Run `sudo iptables-legacy -P FORWARD ACCEPT` once; it applies to this Codespace only.

If you missed this meeting: copy `compose.yaml` and `.github/workflows/ci.yml` from this repo into your own, each as a whole file (`ci.yml` also has the `build_images` and `publish_images` jobs). There is no setting to change. None of this is homework.

Docs: [Compose file reference](https://docs.docker.com/reference/compose-file/), [Services: healthcheck and depends_on](https://docs.docker.com/reference/compose-file/services/), [docker compose up](https://docs.docker.com/reference/cli/docker/compose/up/), [Interpolation and the .env file](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/), [Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

Tasks:

1. A `build_images` job in `.github/workflows/ci.yml`: a matrix over `tasks-api` and `title-normalizer`, `docker build` for each one from its folder. It only builds. Push to `main`.
2. A `compose.yaml` in the repo root: both services, each built from its folder, the ports 5000 and 5001 published. `docker compose up --build` runs both.
3. `tasks-api` reaches `title-normalizer` by its service name. `POST /tasks` with the title `  buy   MILK ` returns `"title":"Buy MILK"` and `"normalized":true`. A wrong result is silent: the task is created and `normalized` is `false`.
4. A `publish_images` job in `ci.yml`: it runs only on a push, not on a pull request, after `build_images`. For both services it pushes the image to `ghcr.io/<owner>/<service>`, tagged with the commit. Push to `main` and find the package. The docs page Working with the Container registry shows how to log in, build and push an image.
5. A healthcheck on `title-normalizer`, and `tasks-api` starts only when it is healthy. `docker compose ps` shows `healthy`.
6. A `smoke_test` job in `ci.yml`: start both services with Compose, post a task, fail the job when `normalized` is not `true`, shut the services down.
7. Move the two published ports and the `NORMALIZER_URL` value out of `compose.yaml` into a `.env` file. `docker compose config` shows the same result as before.

### Meeting 6: protecting main and scanning images

A ruleset on `main` can require pull requests, approvals and named checks, and it can have a bypass list. A scan job checks an image against a database of known flaws and can fail the run; publishing can wait for it. An action can be pinned by commit.

If you missed this meeting: copy `.github/workflows/ci.yml` from this repo into your own as a whole file, and `services/tasks-api/Dockerfile`, `services/tasks-api/.dockerignore`, `services/title-normalizer/Dockerfile` and `services/title-normalizer/.dockerignore` if you have none. The ruleset is a setting, not a file (Settings > Rules > Rulesets, a new branch ruleset): any name, Enforcement status Active, target the default branch, bypass list Repository admin with Always allow, require a pull request with 1 approval, require status checks `run_tests (tasks-api, 3.12)`, `run_tests (title-normalizer, 3.12)`, `build_images (tasks-api)` and `build_images (title-normalizer)`. After the first green scan add `scan_images (tasks-api)` and `scan_images (title-normalizer)`. To switch the ruleset off, set Enforcement status to Disabled. A ruleset needs a public repository on a free account. None of this is homework.

Docs: [About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets), [Troubleshooting required status checks](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks), [Trivy Action README](https://github.com/aquasecurity/trivy-action#readme), [Trivy: container image](https://trivy.dev/docs/latest/guide/target/container_image/), [Trivy: filtering](https://trivy.dev/docs/latest/guide/configuration/filtering/), [Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

Tasks:

1. Read your `publish_images` job and find the package it pushed: the image name and the tag.
2. A ruleset on `main` with the settings above. Open a pull request and read its merge box.
3. A `scan_images` job after `build_images` (a matrix over both services, a Trivy scan of each image that fails on `CRITICAL` and `HIGH` findings that have a fix), with the Trivy action pinned by the commit SHA of a release you looked up. `publish_images` runs only after `scan_images`. Do it on a pull request, add the two `scan_images` checks to your ruleset, then merge.
4. A job `checks_passed` that needs `run_tests`, `build_images` and `scan_images` and fails unless all three succeeded. Require that one name in your ruleset.
5. Pin the other actions in `ci.yml` by the full commit SHA, with the version in a comment.
6. Read your whole `ci.yml` and say aloud what each job does.

### Coming later (planned)

- Quality and security: Dependabot, lint gates. Docs: [Securing your software supply chain](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain).
- Infrastructure as code with Terraform and the Docker provider (no cloud account). Docs: [kreuzwerker/docker provider](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs).
- Monitoring: metrics from the app, Prometheus, Grafana. Docs: [Prometheus overview](https://prometheus.io/docs/introduction/overview/).

aboba