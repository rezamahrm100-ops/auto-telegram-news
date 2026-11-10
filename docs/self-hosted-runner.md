# Independent project: self-hosted runner and GitHub Releases

This fork is a standalone product. It does **not** deploy to, import settings from,
or reuse the databases and credentials of any other project.

## Required runner

Register a Linux self-hosted GitHub Actions runner that can access this repository:
**Settings → Actions → Runners → New self-hosted runner**.

- Add the custom label `auto-telegram-news` to this runner. All project workflows
  target `[self-hosted, auto-telegram-news]`; without that label they remain queued.
- Install Docker Engine + Docker Buildx and Python 3.12 on the runner host.
  The GitHub Actions runner process must be able to access Docker safely.
- Use a dedicated runner instance / OS account and work directory for this project.
  A label routes jobs but does not isolate host files or secrets.
- Do not expose this runner to untrusted pull request code. These workflows
  intentionally have **no** `pull_request` trigger.
- Enable Actions on the fork if GitHub has disabled them.
- Keep GitHub Actions and Release settings under the fork's own repository;
  never reuse credentials from a different project.

## CI: tests and Docker

- A push to `main` runs Python tests.
- Relevant changes to `main` also run a Docker build check.
- Both checks can also be started from **Actions → Run workflow** on `main`.
- Changes are prepared on feature branches and reviewed before merging to `main`.
  Untrusted pull-request builds must not run on the self-hosted runner.

## GitHub Releases

The existing `release-please` configuration is retained. After changes are merged
to `main`, release-please can propose a versioned release pull request.
Review and merge that PR to approve a GitHub Release.

In **Settings → Actions → General**, allow workflow write access and allow
GitHub Actions to create pull requests if needed by release-please.
This is a repository-level setting; the workflow cannot grant it to itself.

To publish a Docker image for a completed GitHub Release:

1. Open **Actions → Publish release image (self-hosted)**.
2. Choose the `main` branch, enter the published release tag (e.g. `v1.8.2`).
3. Click **Run workflow**. The job verifies that the GitHub Release exists,
   checks out exactly that tag, and pushes an image to
   `ghcr.io/rezamahrm100-ops/auto-telegram-news:<release-tag>`.
4. Review the Actions job output and the image under GitHub **Packages**.

An image is not published automatically on unreviewed pushes or pull requests.
Docker image publishing requires repository package write permissions.

## Running the Telegram service (separate from CI)

The runner builds/tests/releases software. The Telegram service itself must run
as a long-lived application, independent of GitHub Actions, with its own
PostgreSQL storage. On the intended deployment host:

```bash
git clone https://github.com/rezamahrm100-ops/auto-telegram-news.git
cd auto-telegram-news
cp .env.example .env
chmod 600 .env
# Set strong APP_SECRET_KEY, ADMIN_PASSWORD, POSTGRES_PASSWORD,
# and your own Telegram credentials in .env, then:
docker compose up -d --build
```

Do not commit `.env`, Telegram sessions, database dumps, or API tokens to GitHub.
The existing Compose file publishes port 8000 on the host; restrict access using
the host firewall or a properly configured reverse proxy before Internet exposure.
Keep regular backups of the independent PostgreSQL volume and `./data` directory.

This PR configures release automation but does **not** register a runner,
start a Docker container, or deploy a live Telegram bot.
