# Gotchas

Things that bite consumers. Scan before architecting around the library.

## One Container per process

`sqlstore` is **not** safe to share across processes. Inside a single process you can have multiple Clients sharing one Container, but don't have two processes opening the same SQLite file. For Postgres backends concurrent access is fine, but design for one Container struct per process.

## `Connect()` returns before "online"

`Connect()` returns once the websocket dial+handshake has started, not once the session is authenticated. Use `*events.Connected` in your handler to know you can safely send. The library's `EnableAutoReconnect` defaults to `true`, so transient drops recover automatically. Set `AutoReconnectHook func(error) bool` to customize / cap retries.

## `AutoTrustIdentity` defaults to `true`

Set `client.AutoTrustIdentity = false` (after `NewClient`, before `Connect`) if you need to verify Signal safety-number changes before continuing a conversation. With it off, you'll receive `*events.IdentityChange` and the library will refuse to encrypt to a peer whose identity key changed until you re-trust it. With it on, the library trusts new identity keys silently.

## AD-JIDs and `ErrRecipientADJID`

A JID with `Device > 0` targets a specific device of a user. `SendMessage` will refuse such a target unless you pass `SendRequestExtra{Peer: true}` (sending an app-protocol message to a specific device). Normal app code sends to the bare user JID (`Device == 0`) and the library fans out to all of the recipient's devices.

## LID JIDs (`@lid`)

WhatsApp is rolling out LID-format JIDs (`HiddenUserServer = "lid"`) for privacy. You may see `events.Message.Info.Sender` with `Server == "lid"`. Treat LID JIDs as opaque identifiers; the library's `ContactStore` maps LID ↔ phone JIDs internally.

## Pre-key replenishment is automatic

The Client keeps the server stocked with one-time pre-keys (default 50, replenishes below 5). Don't try to manage this manually — there's no public API for it and there shouldn't need to be.

## `HistorySync` is your only chance at history

The library does **not** persist conversation history. On a fresh pair you'll get `*events.HistorySync` events containing recent messages, but after that, anything you want to keep you must store yourself. The sync includes contact names — even if you don't care about message history, you may want to consume `HistorySync` to seed the contact store with PushName/FullName.

## Sentinel errors

From `errors.go`. Use `errors.Is`:

- `ErrClientIsNil` — calling a method on a nil Client.
- `ErrNotLoggedIn` — Client has no paired device.
- `ErrNotConnected` — Client isn't connected.
- `ErrIQTimedOut` — server didn't respond to an IQ within the timeout.
- `ErrQRAlreadyConnected` — called `GetQRChannel` after `Connect`.
- `ErrRecipientADJID` — see "AD-JIDs" above.

## Don't import `binary/`

The `binary/` package is the internal wire-protocol encoder. Consumer code should only touch the main `whatsmeow` package plus `types`, `types/events`, `store`, `store/sqlstore`, `util/log`, and the `proto/wa*` protobufs you need (primarily `proto/waE2E`).

## `RemoveEventHandler` from a handler deadlocks

Always wrap in `go func() { ... }()`. See [events.md](events.md).

## Status/Newsletter sends

Sending to `types.StatusBroadcastJID` (status updates) works but is flagged experimental in the README and may not work for large contact lists. Newsletter sends use `Server: "newsletter"` JIDs and have separate upload paths (`client.UploadNewsletter`, `upload.go:164`).

## Logging out vs disconnecting

- `client.Disconnect()` — closes the socket. Reversible: `Connect()` resumes.
- `client.Logout(ctx)` — tells the server to unlink the device and wipes the local Device. Not reversible; you'll need to re-pair.

## Versioning

There's no semver discipline. Pin a commit hash in `go.mod` (`go.mau.fi/whatsmeow v0.0.0-YYYYMMDDhhmmss-abc123def456`). Read commit messages before upgrading — protobuf regenerations and renames happen.
