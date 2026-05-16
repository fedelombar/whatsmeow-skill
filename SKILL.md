---
name: using-whatsmeow
description: Use when working with the go.mau.fi/whatsmeow Go library — building a WhatsApp multidevice client, pairing via QR or phone code, sending messages, handling incoming events, managing groups, or constructing JIDs. Trigger on imports of go.mau.fi/whatsmeow, store/sqlstore, types/events, or waE2E.
---

# Using whatsmeow

A reference for consuming `go.mau.fi/whatsmeow` from external Go projects. Documents the library's real API — does not prescribe wrapper conventions.

## Module facts

- Module: `go.mau.fi/whatsmeow`
- Minimum Go: `1.25.0` (see go.mod)
- License: MPL-2.0
- Godoc: https://pkg.go.dev/go.mau.fi/whatsmeow
- API stability: no formal guarantee. Pin a specific commit/tag; expect occasional breaking changes.

## Quickstart

This is the canonical example, taken verbatim from `client_test.go` in the whatsmeow repo. It pairs (first run, QR), reconnects (subsequent runs), receives messages, and shuts down on `Ctrl+C`. For headless servers where QR scanning isn't practical, swap the QR block for phone-code pairing — see [connecting.md](connecting.md).

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"

    _ "github.com/mattn/go-sqlite3" // REQUIRED: blank-import the DB driver for your sqlstore dialect

    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/store/sqlstore"
    "go.mau.fi/whatsmeow/types/events"
    waLog "go.mau.fi/whatsmeow/util/log"
)

func eventHandler(evt interface{}) {
    switch v := evt.(type) {
    case *events.Message:
        fmt.Println("Received a message!", v.Message.GetConversation())
    }
}

func main() {
    dbLog := waLog.Stdout("Database", "DEBUG", true)
    ctx := context.Background()
    container, err := sqlstore.New(ctx, "sqlite3", "file:examplestore.db?_foreign_keys=on", dbLog)
    if err != nil {
        panic(err)
    }
    // For multiple sessions, persist the JID after PairSuccess and call .GetDevice(jid) instead.
    deviceStore, err := container.GetFirstDevice(ctx)
    if err != nil {
        panic(err)
    }
    clientLog := waLog.Stdout("Client", "DEBUG", true)
    client := whatsmeow.NewClient(deviceStore, clientLog)
    client.AddEventHandler(eventHandler)

    if client.Store.ID == nil {
        // No stored ID → first login. Connect, then read QR codes from the channel.
        qrChan, _ := client.GetQRChannel(context.Background())
        if err := client.Connect(); err != nil {
            panic(err)
        }
        for evt := range qrChan {
            if evt.Event == "code" {
                fmt.Println("QR code:", evt.Code) // render with qrterminal etc.
            } else {
                fmt.Println("Login event:", evt.Event)
            }
        }
    } else {
        // Already paired → just reconnect.
        if err := client.Connect(); err != nil {
            panic(err)
        }
    }

    c := make(chan os.Signal, 1)
    signal.Notify(c, os.Interrupt, syscall.SIGTERM)
    <-c
    client.Disconnect()
}
```

## Architecture (mental model)

- **`sqlstore.Container`** — owns the database and one or more `store.Device`s. One container per process.
- **`store.Device`** — credentials, keys, and identity for one logged-in WhatsApp account (one phone link).
- **`whatsmeow.Client`** — the live connection. Wraps a Device, exposes `Connect`, `SendMessage`, `AddEventHandler`, group/user methods, etc.
- **`waE2E.Message`** (from `go.mau.fi/whatsmeow/proto/waE2E`) — the protobuf message type you build to send anything: text, media, replies, reactions.
- **`types/events.*`** — event structs your handler will type-switch on (`*events.Message`, `*events.Connected`, `*events.QR`, etc.).

## Navigation

| Topic                                                                       | File                                     |
| --------------------------------------------------------------------------- | ---------------------------------------- |
| Container + device + client bootstrap, QR pairing, phone pairing, reconnect | [connecting.md](connecting.md)           |
| `SendMessage`, JIDs, text/media/reply/reaction/edit/revoke, mentions        | [sending.md](sending.md)                 |
| `AddEventHandler`, event type catalog, handler gotchas                      | [events.md](events.md)                   |
| Group operations, contact store, business profiles                          | [groups-contacts.md](groups-contacts.md) |
| Cross-cutting gotchas (LID JIDs, DB-per-process, identity trust, etc.)      | [gotchas.md](gotchas.md)                 |

## Quick reference: most-used Client methods

Source paths are relative to the whatsmeow module root.

| Operation           | Method                                                              | Source                           |
| ------------------- | ------------------------------------------------------------------- | -------------------------------- |
| Construct client    | `whatsmeow.NewClient(deviceStore, log)`                             | `client.go`                      |
| Open DB             | `sqlstore.New(ctx, dialect, dsn, log)`                              | `store/sqlstore/container.go:46` |
| Load device         | `container.GetFirstDevice(ctx)` / `GetDevice(ctx, jid)`             | `store/sqlstore/container.go`    |
| QR pairing          | `client.GetQRChannel(ctx)`                                          | `qrchan.go:162`                  |
| Phone code pairing  | `client.PairPhone(ctx, phone, showNotif, clientType, displayName)`  | `pair-code.go:92`                |
| Connect             | `client.Connect()` / `client.ConnectContext(ctx)`                   | `client.go:479`                  |
| Subscribe to events | `client.AddEventHandler(func(evt any) { ... })`                     | `client.go:763`                  |
| Send                | `client.SendMessage(ctx, jid, *waE2E.Message, ...SendRequestExtra)` | `send.go:185`                    |
| Upload media        | `client.Upload(ctx, plaintext, MediaImage \| MediaVideo \| ...)`    | `upload.go:69`                   |
| Group info          | `client.GetGroupInfo(ctx, jid)`                                     | `group.go:591`                   |
| Joined groups       | `client.GetJoinedGroups(ctx)`                                       | `group.go:503`                   |
| Business profile    | `client.GetBusinessProfile(ctx, jid)`                               | `user.go:413`                    |
| Disconnect          | `client.Disconnect()`                                               | `client.go`                      |

## DB driver imports

`sqlstore.New` takes a `database/sql` dialect string. You **must** also blank-import the matching driver:

| Dialect      | Blank import                                                                              |
| ------------ | ----------------------------------------------------------------------------------------- |
| `"sqlite3"`  | `_ "github.com/mattn/go-sqlite3"`                                                         |
| `"postgres"` | `_ "github.com/jackc/pgx/v5/stdlib"` (and use `"pgx"` dialect) or `_ "github.com/lib/pq"` |

For an existing `*sql.DB`, use `sqlstore.NewWithDB(db, dialect, log)` and then `container.Upgrade(ctx)`.

## When NOT to use this library

Per the whatsmeow README, the following are explicitly **not** implemented — don't generate code that pretends they are:

- Broadcast list messages (not supported on WhatsApp Web either).
- Voice/video calls.

Status messages work but are flagged experimental.

## Logging

`waLog.Stdout(module, minLevel, color)` returns a `waLog.Logger`. Levels: `"DEBUG"`, `"INFO"`, `"WARN"`, `"ERROR"`, or `""` for all. Passing `nil` to `NewClient` is valid (no-op logger).
