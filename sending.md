# Sending messages

`SendMessage`, JID construction, and one canonical example per message kind.

## Signature

```go
func (cli *Client) SendMessage(
    ctx context.Context,
    to types.JID,
    message *waE2E.Message,
    extra ...SendRequestExtra,
) (SendResponse, error)
```
(source: `send.go:185`)

Returns `SendResponse{ID, Timestamp, ServerID, ...}` (`send.go:113`). At most one `SendRequestExtra` may be passed.

`SendRequestExtra` fields (`send.go:142`):

| Field | Use |
|---|---|
| `ID` | Override the auto-generated message ID. |
| `Timeout` | Override the default per-send timeout. |
| `Peer` | Set true to send to a specific device (AD-JID with `Device > 0`). Otherwise hitting an AD-JID returns `ErrRecipientADJID`. |
| `MediaHandle` | Server-side handle from a previous `Upload`, for resends without re-uploading. |

## JIDs

`types/jid.go` defines the JID struct and server constants.

```go
import "go.mau.fi/whatsmeow/types"

userJID  := types.NewJID("15551234567", types.DefaultUserServer) // 15551234567@s.whatsapp.net
groupJID := types.NewJID("120363042000000000", types.GroupServer) // ...@g.us
parsed, err := types.ParseJID("15551234567@s.whatsapp.net")
```

Server constants (`types/jid.go:22-34`):

| Constant | String | Used for |
|---|---|---|
| `DefaultUserServer` | `s.whatsapp.net` | Regular users |
| `GroupServer` | `g.us` | Groups |
| `BroadcastServer` | `broadcast` | Status/broadcast targets |
| `NewsletterServer` | `newsletter` | Channels |
| `HiddenUserServer` | `lid` | LID-format user JIDs (newer privacy-preserving IDs) |
| `BotServer` | `bot` | Meta AI etc. |
| `LegacyUserServer` | `c.us` | Old WhatsApp business JIDs |

Common pre-built JIDs: `types.StatusBroadcastJID` (`status@broadcast`), `types.MetaAIJID`.

## Building messages

`*waE2E.Message` is a protobuf with optional fields. Use `google.golang.org/protobuf/proto` helpers like `proto.String`.

### Text

```go
import "google.golang.org/protobuf/proto"

_, err := client.SendMessage(ctx, userJID, &waE2E.Message{
    Conversation: proto.String("Hello, world!"),
})
```

### Image (and other media)

Upload first, then build the message with the upload metadata:

```go
data, _ := os.ReadFile("cat.jpg")
upload, err := client.Upload(ctx, data, whatsmeow.MediaImage)
if err != nil {
    return err
}
_, err = client.SendMessage(ctx, chatJID, &waE2E.Message{
    ImageMessage: &waE2E.ImageMessage{
        Caption:       proto.String("look at this cat"),
        Mimetype:      proto.String("image/jpeg"),
        URL:           &upload.URL,
        DirectPath:    &upload.DirectPath,
        MediaKey:      upload.MediaKey,
        FileEncSHA256: upload.FileEncSHA256,
        FileSHA256:    upload.FileSHA256,
        FileLength:    proto.Uint64(uint64(len(data))),
    },
})
```

`MediaType` constants (`download.go:43`): `MediaImage`, `MediaVideo`, `MediaAudio`, `MediaDocument`, `MediaHistory`, `MediaAppState`. For each media kind there is a corresponding `waE2E.{Image,Video,Audio,Document}Message`.

For streaming uploads use `client.UploadReader(ctx, reader, tempFile, mediaType)` (`upload.go:104`).

### Reply (quoted message)

```go
import "go.mau.fi/whatsmeow/proto/waCommon"

msg := &waE2E.Message{
    ExtendedTextMessage: &waE2E.ExtendedTextMessage{
        Text: proto.String("agreed"),
        ContextInfo: &waE2E.ContextInfo{
            StanzaID:      proto.String(quotedMessageID),
            Participant:   proto.String(quotedSenderJID.String()),
            QuotedMessage: quotedMessage, // the *waE2E.Message you're replying to
        },
    },
}
_, err := client.SendMessage(ctx, chatJID, msg)
```

### Reaction

```go
reaction := client.BuildReaction(chatJID, senderJID, targetMessageID, "👍")
_, err := client.SendMessage(ctx, chatJID, reaction)
// pass an empty string to remove a reaction.
```
(source: `send.go:528`)

### Edit

```go
edited := client.BuildEdit(chatJID, originalMessageID, &waE2E.Message{
    Conversation: proto.String("typo fixed"),
})
_, err := client.SendMessage(ctx, chatJID, edited)
```
(source: `send.go:593`)

### Revoke / delete-for-everyone

```go
revoke := client.BuildRevoke(chatJID, senderJID, targetMessageID)
// For your own messages, pass types.EmptyJID as senderJID.
_, err := client.SendMessage(ctx, chatJID, revoke)
```
(source: `send.go:513`)

## Mentions

Two requirements — both must be present for the mention to render:

1. The displayed `@<phone>` token in the message body text.
2. The full mentioned JID(s) in `ContextInfo.MentionedJID` as strings.

```go
mention := userJID.String()                // "15551234567@s.whatsapp.net"
msg := &waE2E.Message{
    ExtendedTextMessage: &waE2E.ExtendedTextMessage{
        Text: proto.String("hey @15551234567 check this"),
        ContextInfo: &waE2E.ContextInfo{
            MentionedJID: []string{mention},
        },
    },
}
```

## Typing indicators (`SendChatPresence`)

Signal that you're typing or recording audio in a chat. Required before/after sends in bots that want to feel "alive".

```go
// "typing..."
client.SendChatPresence(ctx, chatJID, types.ChatPresenceComposing, types.ChatPresenceMediaText)
// "recording audio..."
client.SendChatPresence(ctx, chatJID, types.ChatPresenceComposing, types.ChatPresenceMediaAudio)
// stop indicator
client.SendChatPresence(ctx, chatJID, types.ChatPresencePaused, types.ChatPresenceMediaText)
```
(source: `presence.go:130`)

Global online/offline status uses `SendPresence(ctx, types.PresenceAvailable | types.PresenceUnavailable)` (`presence.go:64`). Most accounts need at least one `SendPresence(PresenceAvailable)` after `Connect` to receive incoming chat presence from peers.

## Marking messages as read

```go
err := client.MarkRead(
    ctx,
    []types.MessageID{messageID},   // batch reads in one call
    time.Now(),
    chatJID,
    senderJID,                      // for groups, the participant; for 1:1 chats, types.EmptyJID
)
```
(source: `receipt.go:189`)

Variadic last arg accepts `types.ReceiptTypeRead` (default), `types.ReceiptTypeDelivered`, `types.ReceiptTypeReadSelf`, or `types.ReceiptTypePlayed` (for voice notes).

## Common errors

- `ErrNotLoggedIn` — Client is connected but no paired device. Pair first.
- `ErrRecipientADJID` — sending to a JID with `Device > 0` requires `SendRequestExtra{Peer: true}`. Most app code should send to bare user JIDs (Device == 0).
- `ErrIQTimedOut` — server didn't ack within timeout; check connectivity.
