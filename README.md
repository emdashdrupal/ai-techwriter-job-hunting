# AI Tech Writer Job Hunting

A personal toolkit for AI-assisted job hunting in technical writing, content strategy, information architecture, and documentation roles. It combines:

- **Two LLM system prompts**&thinsp;&mdash;&thinsp;one that tailors your resume to a specific job description, and one that brutally tears it down like a veteran hiring manager.
- **Two n8n workflows**&thinsp;&mdash;&thinsp;one that auto-collects and parses job alert emails (LinkedIn, Indeed) into a Google Sheet, and one that emails you a daily digest of new openings and tracks which ones you've already seen.

## Repository contents

| File | Description |
| --- | --- |
| `resume-builder-prompt.md` | System prompt for an LLM resume generation assistant. Feed it your prior resumes plus a job description, and it extracts the key requirements, writes a custom (non-canned) summary, scores bullet points against JD keywords, and outputs DOCX-ready Markdown tailored to the role. |
| `resume-destroyer-prompt.md` | System prompt for "THE RESUME DESTROYER," a merciless hiring-manager persona. It runs a brutal teardown (including a per-section "BS Factor"), a strategic rebuild with before/after rewrites and ATS optimization, and a competitive reality check. |
| `job-email-collection-and-processing.json` | n8n workflow import: watches an IMAP mailbox for job alert emails, parses job details out of LinkedIn and Indeed emails, and upserts them into a Google Sheet. |
| `generate-job-summaries.json` | n8n workflow import: runs daily at 07:00, collects undigested jobs from the sheet, emails you an HTML digest, and marks those jobs as digested. |

## How the job tracking pipeline works

> [!NOTE] You can use Gmail through a dedicated Gmail node in n8n.

```text
Job alert emails (LinkedIn / Indeed)
        |
        v
IMAP mailbox: Inbox/Career/job-emails
        |
        v
[Workflow 1: Job email collection and processing]
  IMAP trigger (last 50 minutes) -> parse emails (JS) -> upsert into Google Sheet
        |
        v
Google Sheet ("n8n-job-search", Sheet1)  <- job tracker / dedupe store
        |
        v
[Workflow 2: Generate job summary]
  Daily 07:00 schedule -> read sheet -> build HTML digest ->
  send email -> mark jobs as digested
```

### Workflow 1&thinsp;&mdash;&thinsp;Job email collection and processing

Flow: `Email Trigger (IMAP)` &rarr; `GetEmailsList email` &rarr; `Code in JavaScript` &rarr; `Append or update row in sheet`

- **Email Trigger (IMAP):** watches `INBOX/Career/job-emails` for new messages.
- **GetEmailsList email:** fetches messages received in the last ~50 minutes.
- **Code in JavaScript:** detects the source (LinkedIn vs. Indeed vs. unknown), then parses out `job_id`, `job_title`, `company`, `location`, `job_url`, and `received_at`. For LinkedIn, it normalizes URLs to `https://www.linkedin.com/jobs/view/<jobId>/` so the same posting never duplicates.
- **Append or update row in sheet:** upserts into the Google Sheet using `job_id` as the match column, so re-delivered alerts update the existing row instead of creating duplicates.

### Workflow 2&thinsp;&mdash;&thinsp;Generate job summary

Flow: `Schedule Trigger` (07:00 daily) &rarr; `Get row(s) in sheet` &rarr; `Code in JavaScript` &rarr; `If (has_jobs?)` &rarr; **true:** `Send an Email` &rarr; `Get row(s) in sheet1` &rarr; `Code in JavaScript1` &rarr; `Update row in sheet` / **false:** no-op

- Reads all rows and filters to jobs where `digested` is not true, sorted newest first.
- Builds a styled HTML email ("Job Digest&thinsp;&mdash;&thinsp;<date>" with title, company, location, and date per job).
- Sends the digest via SMTP if there are new jobs; otherwise does nothing.
- After sending, re-reads the sheet, filters to the undigested jobs, and sets `digested = true` so they aren't emailed again.

### Expected Google Sheet schema

Sheet name: `Sheet1` (`gid=0`) in a spreadsheet like `n8n-job-search`.

| Column | Description |
| --- | --- |
| `job_id` | Stable job identifier (e.g., LinkedIn job ID). Used for dedupe/upsert matching. |
| `job_title` | Position title from the alert email. |
| `company` | Employer name. |
| `location` | Job location. |
| `job_url` | Canonical link to the posting. |
| `received_at` | When the alert was first received. |
| `digested` | `TRUE` once the job is included in a daily digest. |

## Setup

### Prerequisites

- An [n8n](https://n8n.io) instance (cloud or self-hosted) with the `n8n-nodes-imap` package installed.
   >[!NOTE] You don't need the `n8n-nodes-imap` package if using Gmail.
- IMAP access to the mailbox that receives job alert emails, with a folder like `Inbox/Career/job-emails`.
- An SMTP account for sending the daily digest.
- A Google Service Account with access to your job tracker spreadsheet.

### Import the workflows

1. In n8n, import `job-email-collection-and-processing.json` and `generate-job-summaries.json` (Workflows &rarr; Import from file).
2. Replace the placeholder credential IDs and values with your own:
   - `YOUR_IMAP_CREDENTIAL_ID`&thinsp;&mdash;&thinsp;IMAP account (used by the trigger and list nodes).
   - `YOUR_GOOGLE_SHEET_ID`&thinsp;&mdash;&thinsp;the ID of your Google Sheet.
   - `YOUR_GOOGLEAPI_CREDENTIAL_ID`&thinsp;&mdash;&thinsp;Google Service Account credentials.
   - `YOUR_SMTP_CREDENTIAL_ID`&thinsp;&mdash;&thinsp;SMTP account for the digest email.
   - `your.email@example.com`&thinsp;&mdash;&thinsp;sender/recipient of the digest.
3. Create the Google Sheet with the columns listed above, and share it with the service account.
4. Make sure job alert emails are being delivered to `Inbox/Career/job-emails` (e.g., via Gmail labels/forwarding from LinkedIn and Indeed alerts).
5. Activate both workflows.

### Verify

- Send or forward a test job alert email to the monitored folder; a new row should appear in the sheet within one trigger cycle.
- For the digest, either run the workflow manually from n8n or wait for the 07:00 schedule. You should receive an HTML email listing the undigested jobs, after which they are marked `digested = TRUE` in the sheet.

## Using the LLM prompts

### Resume builder

1. Use `resume-builder-prompt.md` as the system prompt in your LLM of choice.
2. Fill in the placeholders in the "STATIC CONTACT INFO" section (name, address, email, phone, LinkedIn, portfolio, GitHub, education) and the `[X] years of experience` note in the summary guidelines.
3. Provide your current resume(s) for context, then paste the job description.
4. You'll get a tailored, DOCX-ready Markdown resume: custom summary, JD-scored bullets, grouped technical skills, optional AI/open-source sections, and speaking/industry involvement&thinsp;&mdash;&thinsp;all within 1&ndash;2 US Letter pages.

### Resume destroyer

1. Use `resume-destroyer-prompt.md` as the system prompt.
2. Add your personal context where indicated (recent roles, employment gaps, volunteer or self-improvement work).
3. Upload your resume and the job description URL when prompted.
4. You'll get a structured critique: brutal assessment (~40%), strategic reconstruction with before/after rewrites (~40%), and a competitive reality check (~20%).
