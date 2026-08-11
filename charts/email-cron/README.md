# email-cron

A rate-limited, resumable email drip campaign as a Kubernetes CronJob.
Designed for deliverability-friendly warm-up: send a fixed number of emails
per run inside a daily time window, ramping up over time with values-only
changes.

## How it works

- The CronJob fires on `schedule` (default every 30 minutes) around the
  clock. The application checks `window.start`/`window.end` in
  `window.timezone` and exits immediately outside the window, so **the window
  is the knob for send hours** — the schedule rarely changes.
- Throughput = `emailsPerRun` × in-window runs per day.
- A cursor on the PVC records how far through the recipient list the campaign
  is, so each run resumes where the previous one stopped. The cursor is keyed
  by the CSV's checksum: an unchanged list resumes, a replaced list starts a
  fresh campaign.
- `suspend: true` pauses the campaign without uninstalling.

## Container contract

The chart is image-agnostic. The image's entrypoint must accept the args
`["email-cron", "<csv path>"]` and honor:

| Env var | Meaning |
|---|---|
| `EMAILS_PER_RUN` | Max emails to send this run |
| `WINDOW_START` / `WINDOW_END` | Send window (HH:MM, end-exclusive) |
| `SEND_TIMEZONE` | IANA timezone the window is evaluated in |
| `RECIPIENTS_CSV` | Path to the mounted recipient CSV (`email,name` columns) |
| `STATE_DIR` | Directory for cursor state (PVC mount) |
| `SENDGRID_TEMPLATE_ID` | SendGrid dynamic template id |
| `EMAIL_ENVIRONMENT` | Sender environment (`production` for real sends) |
| `SENDGRID_API_KEY` | SendGrid API key (from a Secret) |

## Required values

```yaml
image:
  repository: <your registry>/<your image>
  tag: <pinned tag>
sendgrid:
  templateId: d-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Out-of-band resources

Recipient lists are PII and API keys are secrets — neither belongs in git:

```bash
kubectl create configmap email-recipients --from-file=recipients.csv
kubectl create secret generic sendgrid --from-literal=api-key=<key>
```

## Operations

```bash
# One-off run without waiting for the schedule
kubectl create job --from=cronjob/<release-name> <release-name>-manual-1

# Ramp up: raise emailsPerRun and/or widen window.start/window.end, then
# upgrade the release. Both are values-only changes.
```
