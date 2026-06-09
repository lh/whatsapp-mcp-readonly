# WhatsApp MCP Server — read-only fork

A **read-only** fork of [lharries/whatsapp-mcp](https://github.com/lharries/whatsapp-mcp).
The ability to send WhatsApp messages has been **removed at the source**, in both
the Go bridge and the Python MCP server, so the running binaries cannot message
anyone. You can still search and read your own message history, contacts, and
media.

This fork exists for read-only analysis of your own chats (e.g. summarising large
group conversations with a local LLM) where you want the "can't send" guarantee to
be structural rather than a matter of not calling the send tool.

It connects to your **personal WhatsApp account** via the WhatsApp web multidevice
API (using the [whatsmeow](https://github.com/tulir/whatsmeow) library) and stores
message history locally in SQLite.

## What was changed from upstream

- **Go bridge** (`whatsapp-bridge/main.go`): removed the `POST /api/send` HTTP
  handler, the `sendWhatsAppMessage` function, and the message send/upload path.
  The history-sync request (a control message to `status@s.whatsapp.net` that pulls
  back-history) is kept, since it is required for ingestion and cannot message a
  person. The REST API now binds to `127.0.0.1` only, not all interfaces.
- **Python MCP server** (`whatsapp-mcp-server/`): removed the `send_message`,
  `send_file`, and `send_audio_message` tools and their backing functions, plus the
  outbound-audio conversion helper. The remaining tools are read-only.
- Verified: the string `/api/send` does not appear in the compiled bridge binary.

### Security note (read honestly)

Removing send narrows, but does not by itself eliminate, [the lethal
trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/): a prompt
injection in a message can no longer make WhatsApp send anything, because there is
no send path. However, if you connect this MCP server to a **cloud** LLM, that
model still reads your data and could be coaxed into leaking it through its own
output. For the strongest guarantee, run the read tools against a **local** model
with no outbound network. The bridge itself still needs network — that is the
WhatsApp link — and retains `/api/download` to fetch media you received.

## Installation

### Prerequisites

- Go
- Python 3.11+
- An MCP client (e.g. Claude Desktop or Cursor) — optional, only if you want to use
  the MCP server rather than querying the SQLite database directly
- UV (Python package manager): `curl -LsSf https://astral.sh/uv/install.sh | sh`

### Steps

1. **Clone this repository**

   ```bash
   git clone https://github.com/lh/whatsapp-mcp-readonly.git
   cd whatsapp-mcp-readonly
   ```

2. **Run the WhatsApp bridge**

   ```bash
   cd whatsapp-bridge
   go run main.go
   ```

   The first time you run it, scan the QR code with your WhatsApp mobile app to
   authenticate. After roughly 20 days you may need to re-authenticate.

3. **(Optional) Connect the read-only MCP server**

   Copy the JSON below with the appropriate `{{PATH}}` values:

   ```json
   {
     "mcpServers": {
       "whatsapp": {
         "command": "{{PATH_TO_UV}}",
         "args": [
           "--directory",
           "{{PATH_TO_SRC}}/whatsapp-mcp-readonly/whatsapp-mcp-server",
           "run",
           "main.py"
         ]
       }
     }
   }
   ```

   For **Claude Desktop**, save as `claude_desktop_config.json` in
   `~/Library/Application Support/Claude/`. For **Cursor**, save as `mcp.json` in
   `~/.cursor/`. Then restart the client.

### Windows Compatibility

`go-sqlite3` requires **CGO enabled** and a C compiler. CGO is disabled by default
on Windows, so install a C compiler (e.g. via [MSYS2](https://www.msys2.org/)) and:

```bash
cd whatsapp-bridge
go env -w CGO_ENABLED=1
go run main.go
```

## Architecture

1. **Go WhatsApp Bridge** (`whatsapp-bridge/`): connects to WhatsApp's web API,
   authenticates via QR, and stores message history in SQLite. Two DB files are
   written under `whatsapp-bridge/store/`: `messages.db` (chats + messages) and
   `whatsapp.db` (whatsmeow device session keys — credentials; never share it).
2. **Python MCP Server** (`whatsapp-mcp-server/`): exposes read-only tools over the
   stored data.

## Read-only MCP Tools

- **search_contacts**: Search contacts by name or phone number
- **list_messages**: Retrieve messages with optional filters and context
- **list_chats**: List available chats with metadata
- **get_chat**: Get information about a specific chat
- **get_direct_chat_by_contact**: Find a direct chat with a specific contact
- **get_contact_chats**: List all chats involving a specific contact
- **get_last_interaction**: Get the most recent message with a contact
- **get_message_context**: Retrieve context around a specific message
- **download_media**: Download media from a message and get the local file path

### Media downloading

By default only media metadata is stored locally. Use `download_media` with the
`message_id` and `chat_jid` (shown when printing messages that contain media) to
fetch the file and get its local path.

## Troubleshooting

- **QR code not displaying**: restart the bridge; check your terminal supports it.
- **Already logged in**: the bridge reconnects automatically without a QR code.
- **Device limit reached**: remove a linked device from WhatsApp (Settings > Linked
  Devices).
- **No messages loading**: history can take several minutes after first auth.
- **Out of sync**: delete both `whatsapp-bridge/store/messages.db` and
  `whatsapp-bridge/store/whatsapp.db`, then restart to re-authenticate.

## Credit

Forked from [lharries/whatsapp-mcp](https://github.com/lharries/whatsapp-mcp) by
Luke Harries. Licensed under MIT (see `LICENSE`); upstream copyright retained.
