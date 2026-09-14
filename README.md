# SupportFlow - Automated Customer Email Triage & Ticket Management

This is a production quality portfolio project that shows how to automate a real customer support workflow with Python. It focuses on email processing, customer support automation, structured ticket management, business rules, and a usable dashboard. It is built to look and work like something you would actually ship, not like a tutorial CRUD app.

## Overview

Every day a support team gets a lot of email. Each message has to be read, understood, sorted, prioritized, linked to an order or customer, and turned into a ticket someone can act on. Doing that by hand is slow and inconsistent, especially when urgent issues get buried.

SupportFlow handles the repetitive part for you. It reads incoming email, figures out what the customer needs, turns the message into a structured ticket with a category and priority, pulls out the important details, saves everything in a database, and gives you a dashboard to work from. You stay in control of the reply - the app only drafts it.

## Fictional company

This project is built around a fictional e-commerce company called **Northstar Commerce**. Northstar sells online and gets support email about orders, refunds, cancellations, payments, delivery, product questions, account help, and complaints, plus the usual spam. The system is designed to automate the first stage of their support process so the team can focus on helping people instead of sorting mail.

## Business scenario

Northstar was handling support the manual way. Someone opened the inbox, read each email, tried to guess what the customer wanted, picked a category, guessed how urgent it was, hunted for an order number, copied customer details, created a ticket, set a status, and tried to spot the fires like double charges or security issues. Then they still had to write a reply.

With SupportFlow that flow becomes:

- Customer email comes in
- The system reads and normalizes it
- It classifies the request and checks if it is spam
- It assigns a priority from business rules
- It extracts the order number, customer name, and intent flags
- It checks for duplicates and saves a ticket with a full history
- The dashboard shows new, urgent, and high priority work right away
- An agent opens the ticket, reviews the details, and uses the suggested reply

Example: a customer writes "Where is my order #10482?" - SupportFlow creates an ORDER_STATUS ticket, Medium priority, links order 10482, and shows it in the dashboard. Another writes "I was charged twice" - that becomes a PAYMENT ticket, Urgent, so it stands out immediately.

## What it does

**Reading email**

- Two providers: MockEmailProvider for the offline demo that reads 30 sample `.eml` files, and IMAPEmailProvider for a real inbox over SSL that only looks at unseen messages and never deletes anything.
- Handles real email quirks: plain text and HTML, multipart messages, encoded subjects, missing subjects, attachments saved as metadata only, signature and quoted reply stripping, and broken messages handled per email so one bad message never stops the batch.

**Understanding the message**

- Classification into 12 categories with weighted signals and context, not a single keyword. It looks at phrases, how the message is worded, and how frustrated the customer sounds.
- Priority from clear business rules: Low, Medium, High, or Urgent based on category, urgency phrases, money and security triggers, and sentiment.
- Sentiment as Frustrated, Negative, Positive, or Neutral.
- Extraction of order numbers like #10482, Order #10832, or ORDER-12345, plus customer name and flags for refund, cancellation, payment, or delivery. The sender address from the envelope always wins.

**Tickets and safety**

- Tickets store customer, subject, body, category, priority, status, summary, sentiment, order number, confidence, source, assignee, and a draft reply.
- Duplicate check by Message-ID first, then by a fingerprint of sender plus subject plus normalized body. Spam is caught and ignored without creating a ticket.
- SQLite with parameterized queries only. One bad email is saved as a failure and the rest keeps going.

**Optional AI**

- If you add a Gemini key, it can help with category, priority, summary, and sentiment. The output is strict JSON checked with Pydantic. The model never writes to the database and it cannot overwrite the sender email. If it fails, the system falls back to the rules. No key is needed - everything works offline.

**Desktop dashboard**

- Built with CustomTkinter for a clean, readable interface. Big fonts and clear colors so High and Urgent stand out.
- Dashboard with six cards: Total, New, Open, High, Urgent, and Resolved Today, plus a recent tickets table sorted with urgent first.
- Tickets list with filters for category, priority, status, and assignee, plus search across customer, email, order, subject, or ticket id. Click a header to sort, double click to open.
- Ticket detail with the customer and order info, what was detected, the original email, attachments, full history, and actions to change status or priority, assign someone, add a note, mark resolved or reopen, and work on the reply draft.

## How it fits together

```
Incoming mail
     |
     v
Email Provider - Mock for demo or IMAP for real inbox
     |
     v
Email Parser - plain, HTML, multipart, signatures and quotes removed
     |
     v
EmailProcessor - extractor, classifier, priority, optional Gemini, validator
     |
     v
TicketService - dedupe and history
     |
     v
SQLite - supportflow.db
     |
     v
Desktop Dashboard - stats, list, detail, draft reply
```

Short flow:

```
Email -> parse and normalize -> classify and check spam and sentiment -> extract order and flags -> validate -> save ticket and check duplicates -> show in dashboard -> you review and reply
```

With Gemini:

```
Email -> Gemini -> strict JSON -> Pydantic check -> if it fails, use the rules -> business validation -> SQLite
```

## Categories and priorities

12 categories, all changeable in `config.yaml`: ORDER_STATUS, REFUND, CANCELLATION, PAYMENT, TECHNICAL_SUPPORT, PRODUCT_QUESTION, DELIVERY, COMPLAINT, ACCOUNT, GENERAL, OTHER, SPAM.

- ORDER_STATUS - "Where is my order #10482?"
- REFUND - "The item was damaged, I want my money back"
- CANCELLATION - "Please cancel order #10931"
- PAYMENT - "My card was declined / I was charged twice"
- TECHNICAL_SUPPORT - "Checkout shows a 500 error"
- PRODUCT_QUESTION - "Do you have the blue one in medium?"
- DELIVERY - "When will it arrive?"
- COMPLAINT - "Waiting two weeks with no answer"
- ACCOUNT - "I cannot log in"
- GENERAL - "What are your hours?"
- OTHER - fallback
- SPAM - promos and scams, ignored

Priorities are Low, Medium, High, and Urgent. They come from the whole context, not just a keyword.

- Urgent is for double charges, security or fraud, an account that looks compromised, or something that needs action right now
- High is for refunds, broken items, failed payments, or a missing order that is overdue
- Medium is for order status, delivery questions, or account trouble
- Low is for product questions and general info

Example: "I was charged twice for order 10500" comes in as PAYMENT and Urgent. "What are your delivery times?" comes in as GENERAL and Low.

## Database

The file is `database/supportflow.db` and it creates itself on first run.

- `tickets` - the main ticket rows
- `email_messages` - normalized mail used for duplicate checks
- `ticket_events` - the full timeline for each ticket
- `processing_runs` - each run with counts for scanned, created, duplicates, spam, failed, high, urgent
- `processing_failures` - per-message error reasons
- `attachments` - file name, mime type, and size - files are never opened

All queries use parameters.

## Getting started

You need Python 3.10 or newer. It was built on 3.12.

```bash
# 1. create a virtual environment
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS or Linux
source .venv/bin/activate

# 2. install
pip install -r requirements.txt

# 3. optional - real inbox or AI
# copy .env.example to .env and fill in what you need
```

Example `.env`:

```ini
IMAP_HOST=
IMAP_PORT=993
IMAP_USERNAME=
IMAP_PASSWORD=
IMAP_FOLDER=INBOX

GEMINI_API_KEY=
GEMINI_MODEL=gemini-3.5-flash-lite
```

You can leave it empty and the demo will still run. Keys are never committed and they are scrubbed from logs.

## Demo

The project ships with 30 sample emails in `data/sample/emails`. They cover every category and priority, plus a duplicate pair with the same Message-ID, two spam messages, cases with missing order numbers or subjects, an HTML-only message, a multipart message with an attachment, and a quoted reply thread.

```bash
python run.py --demo
```

It loads the 30 emails offline, parses and classifies them, and prints a short summary like:

```
Processing Complete
========================================
Emails scanned:       30
Tickets created:      27
Duplicates skipped:    1
Spam ignored:          2
Failed:                0
High priority:         8
Urgent:                4
```

It then opens the dashboard. Demo tickets already have a suggested reply filled in. Open any ticket to see it, and you can generate a new one, edit, copy, or discard it.

Other commands:

- `python run.py --demo --headless` - same without the window, handy for the terminal
- `python run.py --recreate --demo` - wipe the database and run fresh
- `python run.py --imap` or `python run.py --imap --headless` - pull from a real inbox
- `python run.py --ai --demo` - turn on Gemini for that run if a key is set

## Configuration

All business settings live in `config.yaml` - company name and support address, inbox folder and batch size, confidence threshold, AI defaults, default ticket status and assignee, the full lists for categories, priorities, statuses and assignees, attachment limits, database path, and log settings.

Secrets stay in `.env`. The file `.env.example` is committed as a template with empty values.

## Tech stack

Python 3.12, SQLite with the standard sqlite3 module, Pydantic 2, PyYAML, python-dotenv, CustomTkinter for the interface, and optional Google Gemini through google-genai. Tests run with pytest.

## A few notes

SupportFlow is a case study, so it makes some intentional choices. It is a desktop app by design, not a full helpdesk SaaS. The demo reads local files and real mail needs real IMAP credentials. The rules are English only and the signal lists are small examples you can extend. Only attachment metadata is stored and no risky files are opened. Replies are drafts on purpose - a person always reviews before anything is sent.
