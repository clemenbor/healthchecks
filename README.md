# Uptime monitoring with GitHub Actions + Healthchecks.io

Free uptime monitoring for the Biodiversity Cell Atlas websites. A scheduled GitHub Actions workflow checks each site every ~5 minutes and reports the result to [Healthchecks.io](https://healthchecks.io), which sends the alerts.

| Site | Healthchecks secret |
|------|---------------------|
| https://portal.biodiversitycellatlas.org | `HC_PING_URL_1` |
| https://biodiversitycellatlas.org | `HC_PING_URL_2` |

## How it works

```
GitHub Actions (cron, ~every 5 min)
   │
   ├─ curl the site (follows redirects, 30 s timeout, 2 retries)
   │
   ├─ HTTP 2xx/3xx ──► ping  https://hc-ping.com/<uuid>        → check is UP
   └─ anything else ─► ping  https://hc-ping.com/<uuid>/fail   → check is DOWN (immediate alert)
```

Healthchecks.io does **not** check the sites itself; it only reacts to pings:

- A `/fail` ping marks the check **down** immediately and sends an alert.
- If no ping arrives within *period + grace* (e.g. the workflow stops running), the check also goes **down**.
- A successful ping after a failure marks it **up** again and sends a recovery alert.

Alerts are only sent on **state changes** (up → down, down → up), not on every run.

## Files

```
.github/workflows/uptime.yml   # the scheduled check
README.md
```

## Setup

### 1. Healthchecks.io

Create one check per site and set:

- **Period:** 10 minutes
- **Grace time:** 10–15 minutes

GitHub's scheduled runs are often delayed, so a generous grace time avoids false alarms. Copy each check's ping URL.

### 2. GitHub secrets

In **Settings → Secrets and variables → Actions → Secrets tab**, click **New repository secret** (not an environment secret) and add:

- `HC_PING_URL_1` – ping URL for the portal check
- `HC_PING_URL_2` – ping URL for the main site check

Never commit ping URLs to the repository: anyone with a ping URL can send fake "up" pings. If one leaks, delete that check in Healthchecks.io, create a new one and update the secret.

### 3. Workflow

The workflow lives in `.github/workflows/uptime.yml` on the `main` branch. Scheduled workflows only run from the default branch.

The cron expression `2-59/5 * * * *` runs at :02, :07, :12, … to avoid the top of the hour, when GitHub is most likely to delay or drop scheduled runs.

The repository is public so that GitHub Actions minutes are free and unlimited. Secrets remain hidden in public repositories.

## Notifications

All notifications come from Healthchecks.io, not GitHub.

- **Immediate down/up alerts:** sent to every integration listed under the project's **Integrations** tab (email, Telegram, Slack, Discord, …). Each extra email address must confirm its integration. Make sure each integration is switched **ON** for both checks.
- **Hourly "still down" reminders:** enabled per user in **Account Settings → Email Reports**. These go to the user's account email, not to the extra email integrations.
- **Team members:** invite people via **Project Settings → Team Access**. Each person must enable hourly reminders in their own account settings, and their address must also be added as an email integration to get the immediate alerts.

## Testing

**Manual run:** go to **Actions → uptime-check → Run workflow**. Each job's log shows a line like:

```
HTTP status for https://portal.biodiversitycellatlas.org: 200
```

Both checks should show a fresh *Last Ping* in Healthchecks.io.

**Alert test:** temporarily change a site URL in the workflow to one that doesn't exist, run it manually, and confirm the "down" email arrives. Revert the change and run again to get the "up" email.

## Adding another site

1. Create a new check in Healthchecks.io (same period/grace settings).
2. Add its ping URL as a new secret, e.g. `HC_PING_URL_3`.
3. Add an entry to the matrix in `uptime.yml`:

```yaml
          - site: "https://new-site.example.org"
            ping_secret: HC_PING_URL_3
```

## Troubleshooting

**Check is down but the site is up.** Open the check's event log in Healthchecks.io:

- **No recent pings:** the workflow isn't running on schedule. Check the Actions tab for runs with the *schedule* trigger, make sure the workflow is enabled and on `main`, and push a small commit to re-register the schedule.
- **Recent `fail` pings:** the workflow reached the site but got an error. The fail ping body and the Actions log show the HTTP code. `000` means no connection or a timeout; `403`/`503` may mean the site blocks GitHub's runners.

**No scheduled runs at all.** The first scheduled run can take up to an hour after the workflow is added or changed. Check https://www.githubstatus.com for Actions incidents.

**Workflow disabled after 60 days.** GitHub disables scheduled workflows in public repos after 60 days without commits. Re-enable it from the Actions tab when GitHub emails you, or make a small commit every month or two.

**No email on a successful run.** Expected: emails are only sent when a check changes state.
