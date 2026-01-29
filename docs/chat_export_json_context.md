# Chat/Channel JSON Export — AI-Friendly Context Report

This report summarizes the JSON export format produced by Telegram Desktop for chats/channels, and provides AI-friendly context for building analysis tools. It is derived directly from the JSON writer implementation and the export data model types. Key references:

* JSON writer and message serialization logic: `export_output_json.cpp`.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L26-L1734】
* JSON writer interface and section list: `export_output_json.h`.【F:Telegram/SourceFiles/export/output/export_output_json.h†L30-L111】
* Export data model types used to shape JSON fields: `export_data_types.h`.【F:Telegram/SourceFiles/export/data/export_data_types.h†L26-L1056】

---

## 1) File layout and top-level structure

The JSON export is written to `result.json`. The file is a **single JSON object** with top-level sections written in order by the writer. When exporting “all data”, this object starts with an `"about"` field and then includes blocks like `"personal_information"`, `"contacts"`, `"sessions"`, `"chats"`, and `"left_chats"` (or only a single chat if a single peer export was requested).【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1112-L1734】

Top-level sections are produced by the following methods:

* `start()` adds `"about"` and opens the root object.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1112-L1129】
* `writePersonal()` → `"personal_information"` object.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1166-L1192】
* `writeContactsList()` → `"contacts"` and `"frequent_contacts"` objects.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1350-L1447】
* `writeSessionsList()` → `"sessions"` and `"web_sessions"` objects.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1449-L1593】
* `writeDialogsStart()`/`writeDialogStart()`/`writeDialogSlice()`/`writeDialogEnd()`/`writeDialogsEnd()` → `"chats"` and `"left_chats"` (or a single chat export).【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1595-L1713】
* `finish()` closes the root object.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1716-L1725】

**Root object (all data export)**
```jsonc
{
  "about": "...",
  "personal_information": { ... },
  "profile_pictures": [ ... ],
  "stories": [ ... ],
  "profile_music": [ ... ],
  "contacts": { "about": "...", "list": [ ... ] },
  "frequent_contacts": { "about": "...", "list": [ ... ] },
  "sessions": { "about": "...", "list": [ ... ] },
  "web_sessions": { "about": "...", "list": [ ... ] },
  "other_data": { ... },
  "chats": { "about": "...", "list": [ ... ] },
  "left_chats": { "about": "...", "list": [ ... ] }
}
```

> Note: in a *single peer export*, `"chats"`/`"left_chats"` wrapping is skipped and the file contains just the selected dialog object with `"messages"`.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1112-L1725】

---

## 2) Dialog (chat/channel) schema

Each dialog is serialized by `writeDialogStart()` and wrapped in `"chats"` or `"left_chats"` lists. The dialog object contains:

* `"name"` (optional; omitted for Saved Messages, Replies, Verification Codes)
* `"type"` (chat/channel type string)
* `"id"` (peer ID, numeric)
* `"messages"` (array of message objects)

Fields and type mapping are determined here:【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1599-L1643】

```jsonc
{
  "name": "Chat Title",        // omitted for special system dialogs
  "type": "public_channel",    // see type mapping below
  "id": 123456789,
  "messages": [ ... ]
}
```

**Dialog type string mapping** (from `DialogInfo::Type`):

* `saved_messages`
* `replies`
* `verification_codes`
* `personal_chat`
* `bot_chat`
* `private_group`
* `private_supergroup`
* `public_supergroup`
* `private_channel`
* `public_channel`【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1609-L1623】

**Dialog list grouping**

Dialogs are split into `"chats"` vs `"left_chats"` depending on `isLeftChannel` and are wrapped in:
```jsonc
{
  "about": "...",
  "list": [ { dialog }, ... ]
}
```
This grouping is handled in `writeChatsStart()` / `writeChatsEnd()` / `validateDialogsMode()`.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1646-L1713】

---

## 3) Message schema (core)

Messages are serialized by `SerializeMessage(...)`. This is the **single most important** reference for an analysis tool. The base message fields always include:

* `"id"` (int)
* `"type"` (`"message"` or `"service"`)
* `"date"` (ISO string)
* `"date_unixtime"` (stringified unix seconds)

And optionally:

* `"edited"` and `"edited_unixtime"`
* `"from"` / `"from_id"`
* `"reply_to_message_id"` / `"reply_to_peer_id"`
* `"via_bot"` / `"forwarded_from"` / etc.
* media-related keys (photo, file, video, audio, etc.)
* service action fields (e.g., `"action": "create_group"`, `"action": "pin_message"`, etc.)

Base assembly and the field push logic are defined here:【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L235-L420】

```jsonc
{
  "id": 123,
  "type": "message",
  "date": "2024-01-01T12:34:56",
  "date_unixtime": "1704112496",
  "from": "User Name",
  "from_id": "user123",
  "reply_to_message_id": 122,
  "reply_to_peer_id": "channel999",
  "...": "..."
}
```

### Message identity fields
* `id` is `Message.id` (int32).【F:Telegram/SourceFiles/export/data/export_data_types.h†L881-L889】
* `from_id` is derived from `PeerId` and is stored as a **string** with a prefix: `"user<id>"`, `"chat<id>"`, or `"channel<id>"`. This is generated by `wrapPeerId()` inside `SerializeMessage`.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L293-L312】

### Dates
* `date` uses ISO timestamp string.
* `date_unixtime` is stored as a **string** of seconds.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L72-L79】

---

## 4) Message text payloads

Text is stored in two possible representations:

1) **Plain string** if text has only a single `TextPart::Text`.
2) **Array of objects** if rich formatting or entities are present.

Serialization is controlled by `SerializeText(...)`, which maps each `TextPart` to an object with `type`, `text`, and a type-specific additional field (`user_id`, `document_id`, `href`, `language`, etc.).【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L146-L224】

```jsonc
// Example: simple
"text": "hello"

// Example: rich
"text": [
  { "type": "plain", "text": "hello", "none": "" },
  { "type": "link", "text": "example.com", "href": "https://example.com" }
]
```

Text part types include: `mention`, `hashtag`, `bot_command`, `link`, `email`, `bold`, `italic`, `code`, `pre`, `text_link`, `mention_name`, `phone`, `cashtag`, `underline`, `strikethrough`, `blockquote`, `bank_card`, `spoiler`, `custom_emoji` and more.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L164-L187】

---

## 5) Message actions (service messages)

`SerializeMessage(...)` contains a large switch that emits `"action"` and associated fields for service messages. Examples:

* `"create_group"` with `"members"`
* `"edit_group_title"` with `"title"`
* `"invite_members"`
* `"pin_message"` with `"message_id"`
* `"phone_call"` with call fields
* `"joined_telegram"` and more

These actions are emitted inside the `v::match(message.action.content, ...)` section.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L400-L420】

**AI tool guidance:** treat `"type": "service"` messages as semantically distinct and filter by `"action"` where possible.

---

## 6) Media and file fields

Media fields are derived from `Data::Media`, `Data::File`, `Data::Image`, etc. The file path values are always **relative paths** (e.g., `"media/photo_123.jpg"`), or placeholder strings if unavailable/filtered. File reference handling is centralized in helper functions in `SerializeMessage(...)` (`pushPath`, `pushPhoto`, etc.).【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L362-L398】

Supporting data structures are defined in `export_data_types.h`:

* `File` (relativePath, size, skipReason)
* `Image` (width/height + file)
* `Document` (file + metadata)
* `Photo`, `SharedContact`, `GeoPoint`, `Venue`, etc.【F:Telegram/SourceFiles/export/data/export_data_types.h†L72-L205】

---

## 7) Reactions

Reactions are serialized inside `SerializeMessage(...)` and include emoji/document IDs and recent reaction metadata. The reaction object fields are appended to the message under `"reactions"` when present. The reaction payload and recent list are assembled in the later portion of `SerializeMessage(...)`.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L1100-L1107】

---

## 8) Recommended AI parsing strategy

### 8.1 Normalize IDs

For reply/author analysis, normalize:

* `from_id` (string `user123` / `chat456` / `channel789`)
* `reply_to_peer_id` (same prefix format)

This lets you join across message chains and filter by actor/target pairs. The `wrapPeerId()` formatting logic establishes this exact format.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L293-L312】

### 8.2 Deal with `"text"` dual shape

Text may be a string or array. A robust parser should:

* If string → treat as plain.
* If array → concatenate `text` parts or keep segmented metadata for entity-aware analysis.

Serialization rules are defined in `SerializeText(...)`.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L146-L224】

### 8.3 Service vs normal messages

Use:

* `"type": "message"` for standard user content.
* `"type": "service"` for actions (membership, edits, pins, etc.).

The `type` is determined in `SerializeMessage(...)`.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L264-L273】

---

## 9) JSON Schema (AI-friendly outline)

Below is a high-level schema outline (not a strict JSON Schema draft, but structured for AI tooling):

```jsonc
Root := {
  "about": string,
  "personal_information"?: PersonalInfo,
  "profile_pictures"?: Userpic[],
  "stories"?: Story[],
  "profile_music"?: Message[],
  "contacts"?: { "about": string, "list": Contact[] },
  "frequent_contacts"?: { "about": string, "list": TopPeer[] },
  "sessions"?: { "about": string, "list": Session[] },
  "web_sessions"?: { "about": string, "list": WebSession[] },
  "other_data"?: object|array,
  "chats"?: { "about": string, "list": Dialog[] },
  "left_chats"?: { "about": string, "list": Dialog[] }
}
```

**Dialog**
```jsonc
Dialog := {
  "name"?: string|null,
  "type": string, // saved_messages, personal_chat, public_channel, ...
  "id": number,
  "messages": Message[]
}
```

**Message (core)**
```jsonc
Message := {
  "id": number,
  "type": "message"|"service",
  "date": string,           // ISO
  "date_unixtime": string,  // unix seconds
  "edited"?: string,
  "edited_unixtime"?: string,
  "from"?: string|null,
  "from_id"?: string,       // "user123" / "chat456" / "channel789"
  "reply_to_message_id"?: number,
  "reply_to_peer_id"?: string,
  "text"?: string | TextPart[],
  "reactions"?: Reaction[],
  ... // media, actions, attachments
}
```

**TextPart**
```jsonc
TextPart := {
  "type": string,   // mention, link, bold, italic, ...
  "text": string,
  // One of:
  "user_id"?: string|number,
  "document_id"?: string,
  "language"?: string,
  "href"?: string,
  "collapsed"?: boolean
}
```

**Action example**
```jsonc
Message (service) := {
  "type": "service",
  "action": "pin_message",
  "message_id": 123,
  ...
}
```

---

## 10) Practical query examples (analysis use-cases)

### Find all messages from user A replying to user B, within date range
Pseudocode:

```pseudo
for dialog in chats.list + left_chats.list:
  for msg in dialog.messages:
    if msg.from_id == "userA"
       and msg.reply_to_message_id exists:
         reply_target = find message with id == msg.reply_to_message_id
         if reply_target and reply_target.from_id == "userB":
           if msg.date_unixtime in [start, end]:
             emit msg
```

Notes:
* A proper implementation should build a map `message_id -> message` per dialog for efficient lookups.
* Use `reply_to_peer_id` if the thread can point to another peer.
* `date_unixtime` is stringified; convert to int for comparisons.【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L72-L79】

---

## 11) Minimum files to provide an AI assistant

If you want to onboard an AI assistant to build tooling against this export format, provide:

1) `Telegram/SourceFiles/export/output/export_output_json.cpp` (JSON schema + message semantics).【F:Telegram/SourceFiles/export/output/export_output_json.cpp†L26-L1734】
2) `Telegram/SourceFiles/export/output/export_output_json.h` (writer interface + section list).【F:Telegram/SourceFiles/export/output/export_output_json.h†L30-L111】
3) `Telegram/SourceFiles/export/data/export_data_types.h` (field semantics + data model types).【F:Telegram/SourceFiles/export/data/export_data_types.h†L26-L1056】

These three files are sufficient to reconstruct a schema model, enumerate all message action variants, and implement a robust JSON parser for analysis and filtering.
