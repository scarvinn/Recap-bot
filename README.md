# chat-digest-bot

Telegram bot that logs group chat messages to Supabase, sends a daily AI-generated digest, and archives each digest in Mem0. Built with n8n.

> **Status:** exported snapshot of a workflow that ran in production for a real client. Client-specific data, IDs and secrets were removed, so the workflow needs configuration before it runs (see [Setup](#setup) and [Known limitations](#known-limitations)).

<!-- Add: screenshot of the workflow, GIF of a digest in a test chat -->
<img width="1624" height="725" alt="image" src="https://github.com/user-attachments/assets/61fdbe08-1407-49b8-9872-0d4a29e199a1" />

## How it works

```
Telegram group ──► n8n (Telegram Trigger) ──► filter ──► Supabase (messages)

Schedule (daily) ──► Supabase (last 24h) ──► aggregate ──► LLM (Groq, Llama 3.3 70B) ──┬─► Telegram (digest)
                                                                                         └─► Mem0 (archive)
```

Two independent flows in one n8n workflow:

1. **Collect.** Telegram Trigger receives every message. An IF node keeps only text messages from group chats (`chat.id < 0`). Each message is stored in Supabase with `text`, `username` and `chat_id`.
2. **Digest.** A Schedule Trigger runs once a day, pulls messages from the last 24 hours, aggregates them and sends them to an LLM with a strict prompt. The result is posted to Telegram as HTML and saved to Mem0.

### Digest prompt

The prompt forces the model to stay factual: no advice, no invented events, empty sections are omitted, usernames always get `@`. Output is Telegram-flavoured HTML (`<b>` only). Sections: main topics, questions and requests, decisions and actions, finance and numbers, problems and conflicts.

## Stack

- [n8n](https://n8n.io) (workflow engine)
- [Supabase](https://supabase.com) (Postgres storage)
- [Groq](https://groq.com) with `llama-3.3-70b-versatile` (digest generation)
- [Mem0](https://mem0.ai) (digest archive)
- Telegram Bot API

## Setup

### 1. Telegram bot

1. Create a bot with [@BotFather](https://t.me/BotFather) and save the token.
2. To let the bot see all group messages, disable privacy mode: `/setprivacy` → *Disable*.
3. Add the bot to your group.

### 2. Supabase

Create the table:

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

### 3. n8n

1. Run n8n (Docker example):

   ```bash
   docker run -d --name n8n --restart unless-stopped \
     -p 5678:5678 \
     -e GENERIC_TIMEZONE=Europe/Kyiv \
     -v n8n_data:/home/node/.n8n \
     docker.n8n.io/n8nio/n8n
   ```

2. Import `workflow.json` (*Workflows → Import from file*).
3. Create credentials for Telegram, Supabase and Groq, and attach them to the nodes.
4. Set the target chat ID in the **Send a text message** node.
5. Configure the **Mem0** HTTP node: add your API key as a Header Auth credential and set `user_id`.

Telegram only delivers webhooks to a public HTTPS URL, so the collector needs a tunnel (Cloudflare Tunnel, ngrok) or a server with a domain. Set `WEBHOOK_URL` accordingly.

### 4. Environment

Never commit real tokens. Keep secrets in n8n credentials. See `.env.example` for variables used by the Docker setup.

## Known limitations

- **All chats go into one digest.** Messages are not grouped by `chat_id`. If the bot is in several groups, they are mixed in a single report.
- **Mem0 is write-only.** Digests are archived there, but the generation step does not read from Mem0 yet, so the digest does not use long-term context.
- **Token limits.** Raw rows are sent to the LLM. On busy chats this can exceed Groq's per-minute token limits. Mapping to `[time] @user: text` would reduce this.
- **Telegram message limit.** Digests over 4096 characters will be rejected, and invalid HTML from the model causes a 400 error.
- **Users without a username** appear as `@null`. A fallback to `first_name` is not implemented.
- **Empty days.** There is no guard for days without messages, so the model may receive an empty log.
- **Privacy.** The bot stores chat messages. Tell chat members that the chat is logged and summarized by an AI.

## Roadmap

- [ ] Group digests by chat
- [ ] Read relevant memories from Mem0 before generating the digest
- [ ] Compact message formatting to cut tokens
- [ ] Skip the digest when there are no messages
- [ ] Split long digests into several Telegram messages

## License

MIT
