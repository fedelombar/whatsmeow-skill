# Event handling

How to subscribe, how to dispatch on event types, and one important deadlock to avoid.

## Subscribe

```go
id := client.AddEventHandler(func(evt any) {
    switch v := evt.(type) {
    case *events.Message:
        log.Printf("msg from %s: %s", v.Info.Sender, v.Message.GetConversation())
    case *events.Connected:
        log.Println("online")
    case *events.LoggedOut:
        log.Println("device was logged out; need to re-pair")
    }
})
// later:
client.RemoveEventHandler(id)
```
(source: `client.go:763`, `:790`)

`AddEventHandlerWithSuccessStatus` (`client.go:770`) is the variant whose handler returns `bool` — return `false` to deregister itself.

## DEADLOCK GOTCHA

**Never call `RemoveEventHandler` synchronously from inside an event handler.** It will deadlock — the handler list is locked while your handler runs (see `client.go:781-783`).

```go
// Wrong:
client.AddEventHandler(func(evt any) {
    if shouldStop(evt) {
        client.RemoveEventHandler(id) // DEADLOCK
    }
})

// Right:
client.AddEventHandler(func(evt any) {
    if shouldStop(evt) {
        go client.RemoveEventHandler(id)
    }
})
```

Or use `AddEventHandlerWithSuccessStatus` and return `false`.

## Event catalog (most-used)

The full set is in `go.mau.fi/whatsmeow/types/events`. Common ones:

| Event | Meaning |
|---|---|
| `*events.QR` | A QR payload is available (also delivered via `GetQRChannel`). |
| `*events.PairSuccess` | Pairing finished; `client.Store.ID` is now set. |
| `*events.PairError` | Pairing failed. |
| `*events.Connected` | Websocket up and authenticated. |
| `*events.Disconnected` | Connection lost; auto-reconnect will fire if enabled. |
| `*events.LoggedOut` | Device was unlinked remotely; you must re-pair. Includes `Reason`. |
| `*events.StreamReplaced` | Another session for this account took over. |
| `*events.Message` | Incoming message. `v.Info` is metadata, `v.Message` is the `*waE2E.Message`. |
| `*events.UndecryptableMessage` | Message arrived but couldn't be decrypted (sender's keys missing/stale). |
| `*events.Receipt` | Delivery / read receipt for a message you sent. `v.Type` indicates kind. |
| `*events.Presence` | Online/offline status of a contact. |
| `*events.ChatPresence` | Typing / recording indicator in a chat. |
| `*events.HistorySync` | Bulk historical messages on first login. Persist if you want history. |
| `*events.AppState` / `*events.AppStateSyncComplete` | Contact list, mute/pin state, blocked users. |
| `*events.GroupInfo` | Group metadata changed (subject, participants, etc.). |
| `*events.JoinedGroup` | You were added to or created a group. |
| `*events.Picture` | A profile or group picture changed. |
| `*events.IdentityChange` | A contact's Signal identity key changed (relevant if `AutoTrustIdentity = false`). |

## Reading message content

`events.Message.Message` is `*waE2E.Message`. Use the getter helpers (they're protobuf-generated, nil-safe):

```go
case *events.Message:
    txt := v.Message.GetConversation()                              // plain text
    if txt == "" {
        txt = v.Message.GetExtendedTextMessage().GetText()          // rich text / replies
    }
    if img := v.Message.GetImageMessage(); img != nil {
        // download with client.Download(ctx, img)
    }
```

`v.Info` carries `Chat`, `Sender`, `ID`, `Timestamp`, `IsFromMe`, `IsGroup`, `PushName`, etc.

## Downloading received media

`client.Download(ctx, msg)` (`download.go:235`) decrypts and returns the raw bytes. The argument is any `DownloadableMessage` — i.e. `*waE2E.{Image,Video,Audio,Document,Sticker}Message`.

```go
case *events.Message:
    if img := v.Message.GetImageMessage(); img != nil {
        data, err := client.Download(ctx, img)
        if err != nil {
            log.Printf("download failed: %v", err)
            return
        }
        ext := strings.TrimPrefix(img.GetMimetype(), "image/") // e.g. "jpeg"
        os.WriteFile(fmt.Sprintf("%s.%s", v.Info.ID, ext), data, 0644)
    }
```

If you've already persisted a media message and want to download it later, reconstruct the typed message (e.g. `*waE2E.ImageMessage`) from the saved fields (URL/DirectPath, MediaKey, FileEncSHA256, FileSHA256, FileLength, Mimetype) and pass it to `Download`. The deprecated `client.DownloadAny(ctx, *waE2E.Message)` does the same dispatch automatically but is on the way out — prefer the typed `Download`.
