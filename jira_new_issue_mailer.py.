import csv
import os
import ssl
import smtplib
import logging
from email.message import EmailMessage
from datetime import datetime
from pathlib import Path

# ================== CONFIG (edit as needed) ==================
EXPORT_DIR = Path(r"\\secure-share\jira_exports")  # folder holding your daily CSV exports
LOG_DIR = Path(r"\\secure-share\jira_exports\logs")
LOG_DIR.mkdir(parents=True, exist_ok=True)
LOG_FILE = LOG_DIR / "jira_new_issue_mailer.log"

# Column headers as they appear in your CSV export
COL_ISSUE_KEY = "Issue key"
COL_SUMMARY = "Summary"
COL_REPORTER = "Reporter"
COL_REPORTER_EMAIL = "Reporter email"
COL_CREATED = "Created"

# Email defaults (pulled from env where possible)
FROM_EMAIL = os.environ.get("AUTOMAIL_FROM", "no-reply@yourbank.local")
REPLY_TO = os.environ.get("AUTOMAIL_REPLYTO", "your.name@yourbank.local")  # you, if you want replies
SMTP_HOST = os.environ.get("SMTP_HOST")  # e.g., "smtp.bank.local"
SMTP_PORT = int(os.environ.get("SMTP_PORT", "25"))
SMTP_USER = os.environ.get("SMTP_USER")  # optional
SMTP_PASS = os.environ.get("SMTP_PASS")  # optional
USE_TLS = os.environ.get("SMTP_USE_TLS", "false").lower() in ("1", "true", "yes")

# Fallback: write .eml drafts here if SMTP is not available
DRAFTS_DIR = EXPORT_DIR / "_drafts"
DRAFTS_DIR.mkdir(parents=True, exist_ok=True)

# Email subject/body (generic)
EMAIL_SUBJECT = "Thanks for logging your request — we’ve got it"
EMAIL_BODY = """Hello,

This is a quick confirmation that we’ve received your request in Jira and it’s now in our intake queue.

What happens next:
• Our team will triage and prioritize items.
• If we need additional details, we’ll reach out on the Jira ticket.
• For urgent production-impacting issues, please also follow the standard escalation process.

You do not need to reply to this email. We’ll keep all updates within Jira.

Best regards,
IT Engineering
"""

# If you prefer one email per REPORTER (listing all of their new issues) set True.
# If False, we send one email per new issue.
ONE_EMAIL_PER_REPORTER = True

# ============================================================

logging.basicConfig(
    filename=LOG_FILE,
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)

def load_csv(path: Path):
    with path.open(newline="", encoding="utf-8-sig") as f:
        reader = csv.DictReader(f)
        rows = [row for row in reader]
    return rows

def pick_latest_two_csvs(folder: Path):
    files = sorted([p for p in folder.glob("*.csv")], key=lambda p: p.name)
    if len(files) < 2:
        raise RuntimeError("Need at least two CSVs (yesterday & today).")
    return files[-2], files[-1]

def extract_issue_keys(rows):
    return {row[COL_ISSUE_KEY].strip() for row in rows if row.get(COL_ISSUE_KEY)}

def group_new_by_reporter(rows, new_keys):
    by_reporter = {}
    for r in rows:
        key = (r.get(COL_ISSUE_KEY) or "").strip()
        if key in new_keys:
            email = (r.get(COL_REPORTER_EMAIL) or "").strip()
            if not email:
                # try reporter field if email missing
                email = (r.get(COL_REPORTER) or "").strip()
            if not email:
                logging.warning(f"Skipping {key}: no reporter email.")
                continue
            by_reporter.setdefault(email, []).append({
                "key": key,
                "summary": (r.get(COL_SUMMARY) or "").strip(),
                "created": (r.get(COL_CREATED) or "").strip(),
                "reporter": (r.get(COL_REPORTER) or "").strip(),
            })
    return by_reporter

def build_message(to_addr, subject, body, list_of_issues=None):
    msg = EmailMessage()
    msg["From"] = FROM_EMAIL
    msg["To"] = to_addr
    if REPLY_TO:
        msg["Reply-To"] = REPLY_TO
    msg["Subject"] = subject

    final_body = body
    if list_of_issues:
        final_body += "\n\nNew issues logged by you since yesterday:\n"
        for item in sorted(list_of_issues, key=lambda x: x["key"]):
            line = f"• {item['key']} — {item['summary']} (Created: {item['created']})"
            final_body += line + "\n"
        final_body += "\nWe’ll keep you updated in Jira.\n"

    msg.set_content(final_body)
    return msg

def try_send_via_smtp(msg: EmailMessage):
    if not SMTP_HOST:
        return False, "SMTP_HOST not set"
    try:
        if USE_TLS:
            context = ssl.create_default_context()
            with smtplib.SMTP(SMTP_HOST, SMTP_PORT, timeout=20) as server:
                server.starttls(context=context)
                if SMTP_USER and SMTP_PASS:
                    server.login(SMTP_USER, SMTP_PASS)
                server.send_message(msg)
        else:
            with smtplib.SMTP(SMTP_HOST, SMTP_PORT, timeout=20) as server:
                if SMTP_USER and SMTP_PASS:
                    server.login(SMTP_USER, SMTP_PASS)
                server.send_message(msg)
        return True, "sent"
    except Exception as e:
        return False, str(e)

def save_as_eml(msg: EmailMessage, suffix: str = ""):
    ts = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
    safe_to = (msg["To"] or "unknown").replace("@", "_at_").replace(">", "").replace("<", "")
    name = f"draft_{ts}_{safe_to}{suffix}.eml"
    path = DRAFTS_DIR / name
    with path.open("wb") as f:
        f.write(bytes(msg))
    return path

def main():
    try:
        prev_csv, today_csv = pick_latest_two_csvs(EXPORT_DIR)
        logging.info(f"Comparing {prev_csv.name} -> {today_csv.name}")

        prev_rows = load_csv(prev_csv)
        today_rows = load_csv(today_csv)

        prev_keys = extract_issue_keys(prev_rows)
        today_keys = extract_issue_keys(today_rows)
        new_keys = today_keys - prev_keys

        logging.info(f"Found {len(new_keys)} new issues.")

        if not new_keys:
            logging.info("No new issues. Done.")
            return

        by_reporter = group_new_by_reporter(today_rows, new_keys)

        if ONE_EMAIL_PER_REPORTER:
            for reporter_email, issues in by_reporter.items():
                msg = build_message(
                    to_addr=reporter_email,
                    subject=EMAIL_SUBJECT,
                    body=EMAIL_BODY,
                    list_of_issues=issues,
                )
                ok, info = try_send_via_smtp(msg)
                if ok:
                    logging.info(f"Email SENT to {reporter_email}: {len(issues)} issues.")
                else:
                    p = save_as_eml(msg, suffix="_group")
                    logging.warning(f"SMTP unavailable ({info}). Draft saved: {p}")
        else:
            for reporter_email, issues in by_reporter.items():
                for it in issues:
                    msg = build_message(
                        to_addr=reporter_email,
                        subject=EMAIL_SUBJECT,
                        body=EMAIL_BODY,
                        list_of_issues=[it],
                    )
                    ok, info = try_send_via_smtp(msg)
                    if ok:
                        logging.info(f"Email SENT to {reporter_email}: {it['key']}")
                    else:
                        p = save_as_eml(msg, suffix=f"_{it['key']}")
                        logging.warning(f"SMTP unavailable ({info}). Draft saved: {p}")

        logging.info("Done.")
    except Exception as e:
        logging.exception(f"Failure: {e}")
        raise

if __name__ == "__main__":
    main()
