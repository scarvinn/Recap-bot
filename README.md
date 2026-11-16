# Recap Bot — Telegram chat digests with n8n

An n8n workflow for collecting Telegram group messages in Supabase and generating daily summaries with Groq. It includes a branch intended to archive summaries in Mem0.

> **Status:** sanitized export of a workflow used for a client, not a ready-to-run template. Client-specific configuration was removed. The export also has unfinished data mappings described below. Importing it and adding credentials alone is insufficient. This repository contains documentation and the workflow snapshot, not a hosted bot.

<img width="1624" height="725" alt="Recap Bot workflow in the n8n editor" src="https://github.com/user-attachments/assets/61fdbe08-1407-49b8-9872-0d4a29e199a1" />

## Purpose and architecture

The goal is to turn a busy group chat into a short report of topics, questions, decisions, numbers and problems, with participants attributed by username.

```text
Telegram messages → group/text filter → Supabase messages

Daily schedule → Supabase messages + employee_logs → aggregate → merge
    → JavaScript mapping → Groq LLM → Telegram summary
                                  → Mem0 archive request
```

The collector and scheduled digest are separate flows in the same export. The collector keeps text messages from groups (`chat.id < 0`) and writes `text`, `username` and `chat_id`. The schedule is set to **18:15**; its actual timezone depends on the n8n instance/workflow configuration. Both database queries select records from the preceding 24 hours.

The LLM uses `llama-3.3-70b-versatile` through Groq at temperature 0.1. Its Ukrainian prompt asks for factual summaries, omitted empty sections, `@username` attribution and Telegram HTML formatting. These are instructions to the model, not guarantees that every output will be factual or valid HTML.

## Repository contents

| File | Contents |
| --- | --- |
| [`RECAP BOT.json`](RECAP%20BOT.json) | Importable n8n workflow snapshot, with configuration and mapping gaps. |
| `README.md` | Architecture, setup requirements and limitations. |

There is currently no `.env.example`, Docker Compose file, database migration or automated test suite.

## Inspect the workflow

1. Start an n8n instance. For a local editor:

   ```sh
   docker run -d --name n8n --restart unless-stopped \
     -p 127.0.0.1:5678:5678 \
     -e GENERIC_TIMEZONE=Europe/Kyiv \
     -v n8n_data:/home/node/.n8n \
     docker.n8n.io/n8nio/n8n
   ```

   For a reproducible deployment, pin a tested n8n version. This repo does not currently specify one.

2. Open the editor at `http://localhost:5678` and import **`RECAP BOT.json`** using **Workflows → Import from file**.
3. Keep the workflow inactive while inspecting and configuring it. The exported snapshot has `active: false`.
4. Create Telegram, Supabase and Groq credentials and attach them to the corresponding nodes. Credential references in the export are not usable API keys.
5. Use a test group and sample messages to validate the steps below before activating the schedule.

Telegram webhooks require a public HTTPS endpoint. A local collector needs a tunnel or a server with a domain, plus an appropriate `WEBHOOK_URL` in the n8n environment. The Docker command above only opens a local editor; it does not configure HTTPS or webhook delivery.

## Configuration and mapping work required

### Telegram collection and delivery

Create a bot with [BotFather](https://t.me/BotFather), add it to a test group and configure privacy mode if it must receive ordinary group messages. Set the destination in **Send a text message**; its exported `chatId` expression is empty. Review the **If** node's removed chat-ID condition as well as its group/text checks.

### Supabase tables

The collector writes to `messages`. This schema covers the fields used by that flow:

```sql
create table messages (
  id         bigint generated always as identity primary key,
  created_at timestamptz not null default now(),
  chat_id    bigint not null,
  username   text,
  text       text
);

create index on messages (created_at);
```

The digest also queries **`employee_logs`**. Its schema and ingestion flow are not included. To run the export, either supply that data source and schema or adapt the digest to use only `messages`. Configure database access for the n8n credential according to your deployment.

### LLM input

The aggregate nodes produce `chatdataClient` and `chatdataEmloyee`. The JavaScript node maps them to `clientData` and `emploeeData` (with the apparent client/employee labels reversed), while the LLM prompt reads **`$json.messages`**. Reconcile those fields before testing generation; the export does not currently pass its collected logs into the prompt's expected field.

### Mem0 archive

**HTTP Request1** has a blank token, empty `user_id` and an incomplete JSON body with no `content` value. Configure authentication using n8n credentials, assign the intended user scope and map the generated summary into a valid request body. The archive branch is write-only; generation does not retrieve memories.

Keep service tokens in credentials rather than committing them to the workflow.

## Validation checklist

- Send a sample group text message and verify one row with the right text, username and chat ID.
- Confirm that non-text and private-chat messages are filtered as intended.
- Run the database and mapping nodes manually; verify the LLM receives the actual sample logs.
- Generate a summary and compare every statement against those logs.
- Test Telegram delivery and Mem0 archival independently, including API failures.
- Verify empty-day behavior, long messages and timezone settings before activating the schedule.

These checks are suggested steps; this repository does not include execution evidence or automated coverage for the sanitized export.

## Known limitations

- **Chats are combined.** The database queries do not group or filter by `chat_id`; multiple groups can end up in one report. Scope queries and delivery per group before using this with separate audiences.
- **Missing input guard.** There is no explicit skip for days with no messages.
- **Token limits.** Raw aggregated rows can exceed model context or service token limits. Compact formatting and chunking are not implemented.
- **Telegram output limits.** There is no splitting at Telegram's 4096-character limit or validation of model-generated HTML.
- **Missing usernames.** There is no implemented fallback to first name or another stable label.
- **Rolling time window.** Each run reads the preceding 24 hours rather than tracking a successful processing watermark; delayed runs can leave gaps or duplicate coverage.
- **Operational resilience.** The export does not define explicit retry, failure-notification or duplicate-message handling flows.
- **Data handling.** Messages are stored in Supabase and sent to an external model provider; generated summaries are intended for Mem0. Explain this to participants and define access and retention rules for your deployment.

## Roadmap

- [ ] Make the sanitized export runnable with consistent field mappings.
- [ ] Provide the missing schema/configuration examples and pin a tested n8n version.
- [ ] Generate and deliver separate digests per chat.
- [ ] Skip empty logs, compact inputs and split long Telegram messages.
- [ ] Add sample input/output and repeatable workflow checks.
- [ ] Retrieve relevant memories if long-term context is needed.

