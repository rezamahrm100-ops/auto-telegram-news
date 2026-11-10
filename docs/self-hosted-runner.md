# Self-hosted GitHub Actions and releases

Auto Telegram News is a standalone product with its own source code, database,
runtime configuration, and release pipeline. No other project's secrets or data
are used.

## Using an existing self-hosted runner

All workflows now use `runs-on: self-hosted`, matching a registered runner that
is accessible to this repository **without any custom label**.

- If your runner is already registered and online for this repository, GitHub
  automatically picks up queued jobs. No manual workflow trigger is necessary.
- A runner registered only in another repository is not automatically available
  here. It must be registered for this repository or shared through an
  organization runner group.
- Installing or registering a runner on your personal computer cannot be done
  by committing a workflow: it requires host-side authorization and network
  access to your computer.
- The runner host should have Docker Engine + Docker Buildx and enough disk,
  RAM, and compute for building the AI-enabled Docker image. Python 3.12 is set
  up by actions/setup-python for the tests.
- A dedicated runner process / OS account with a restricted workspace is
  strongly recommended. Labels route jobs, but **do not** sandbox their
  filesystem or protect secrets.
- These workflows do not run untrusted `pull_request` code on your machine.
  Do not merge untrusted code to `main`.

## Automatic build and release process

1. A merged push to `main` triggers the test workflow.
2. Changes to Docker-related paths trigger the Docker build-check workflow.
3. `release-please` runs after pushes to `main`, creating a release proposal
   PR when version-relevant changes are found. The release PR is a deliberate
   approval gate. GitHub may require the repository-level setting allowing
   Actions to create PRs.
4. Once a release PR is merged, `release-please` publishes a GitHub Release
   and its workflow automatically builds and pushes the corresponding image to
   `ghcr.io/rezamahrm100-ops/auto-telegram-news:<release-tag>`.
5. `publish-image.yml` also handles independently published GitHub Releases,
   plus a `workflow_dispatch` retry option. Release events generated using
   the default GITHUB_TOKEN may not trigger a second workflow; the built-in
   release-please publish job handles that case without relying on this event.

There is no automatic deployment to a personal computer from GitHub. The build
and release workflows are independent of the long-lived Telegram service.

## Long-lived Telegram service

The Telegram service needs an always-on host and its own PostgreSQL database.
The project's Docker Compose file starts the app and PostgreSQL when invoked
on the deployment host, but the GitHub runner is not inherently the deployment
host.

```bash
git clone https://github.com/rezamahrm100-ops/auto-telegram-news.git
cd auto-telegram-news
cp .env.example .env
chmod 600 .env
# Set APP_SECRET_KEY, ADMIN_PASSWORD, POSTGRES_PASSWORD and Telegram credentials.
docker compose up -d --build
```

The existing Compose file exposes port 8000; keep this behind firewall/reverse
proxy restrictions. Never commit `.env`, Telegram user sessions, database
backups, or API keys to the public fork. Back up the database volume and media.

## Troubleshooting queued jobs

If jobs remain queued with no runner name, the runner is offline, unavailable
to this repository, busy, or not accepting jobs. GitHub workflows cannot
bootstrap a runner on an unrelated personal computer without host access.
Check the repository's **Settings → Actions → Runners** page.
