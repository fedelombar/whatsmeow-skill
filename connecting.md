# Connecting & pairing

Bootstrap flow, the two pairing methods (QR vs phone code), reconnection behaviour, and shutdown.

## The bootstrap chain

```
sqlstore.New ──► Container ──► (Get|New)Device ──► whatsmeow.NewClient ──► Connect
```

### sqlstore.Container

```go
// New DB file. Auto-runs schema migrations.
container, err := sqlstore.New(ctx, "sqlite3", "file:store.db?_foreign_keys=on", logger)

// Existing *sql.DB.
container, err := sqlstore.NewWithDB(db, "sqlite3", logger)
err = container.Upgrade(ctx) // run migrations manually
```

The `?_foreign_keys=on` query arg is important for SQLite — keys are NOT enabled by default and the schema relies on them.

### Getting a Device

```go
// Single-account use:
device, err := container.GetFirstDevice(ctx)   // creates a fresh device row if none exist

// Multi-account use — persist the JID after *events.PairSuccess, then:
device, err := container.GetDevice(ctx, jid)   // nil device if not found
// or enumerate:
devices, err := container.GetAllDevices(ctx)
```

`container.NewDevice()` returns an in-memory device that gets persisted automatically once pairing succeeds.

### Constructing the Client

```go
client := whatsmeow.NewClient(device, logger) // logger may be nil
```

The Client uses internal sub-loggers for `recv` and `send` streams.

## Pairing flow A — QR (linking a phone)

Used when the user scans a QR code from WhatsApp on their phone (Linked Devices screen).

```go
if client.Store.ID == nil {                 // not yet paired
    qrChan, _ := client.GetQRChannel(ctx)   // must be called BEFORE Connect
    if err := client.Connect(); err != nil {
        return err
    }
    for evt := range qrChan {
        switch evt.Event {
        case "code":
            // evt.Code is the raw QR payload — render with qrterminal,
            // qrencode, or any QR library.
            renderQR(evt.Code)
        case "success":
            // Paired. Channel will close shortly after.
        case "timeout", "err-client-outdated", "err-scanned-without-multidevice":
            return fmt.Errorf("qr pairing failed: %s", evt.Event)
        }
    }
} else {
    err := client.Connect()
}
```

`QRChannelItem.Event` values are defined in `qrchan.go:37-49`.

## Pairing flow B — phone code

Used when QR isn't practical (server, no display). User enters an 8-character code into WhatsApp.

```go
err := client.Connect()
if err != nil {
    return err
}
linkingCode, err := client.PairPhone(
    ctx,
    "15551234567",                          // E.164 without '+'
    true,                                   // showPushNotification on the phone
    whatsmeow.PairClientChrome,             // PairClientType
    "MyApp on Linux",                       // display name shown on the phone
)
// Tell the user: "Open WhatsApp → Linked Devices → Link with phone number → enter <linkingCode>"
```

`PairClientType` values: `PairClientChrome`, `PairClientFirefox`, `PairClientSafari`, `PairClientEdge`, `PairClientOpera` (`pair-code.go`).

## Connection semantics

- `Connect()` is `ConnectContext(BackgroundEventCtx)` — it dials the websocket and returns. It returns `nil` once the handshake is started, not when the session is fully ready.
- "Fully online" is signalled by `*events.Connected` in your handler.
- `EnableAutoReconnect` is **true by default** on a new Client. The library will reconnect on transient failures.
- `InitialAutoReconnect` controls whether a *failed initial* connection also enters the auto-reconnect loop (`client.go:510`).
- `AutoReconnectHook func(error) bool` — set to customize backoff or stop retrying.
- Sentinel errors of interest: `ErrClientIsNil`, `ErrNotLoggedIn`, `ErrNotConnected`, `ErrIQTimedOut`, `ErrQRAlreadyConnected` (see `errors.go`).

## Shutdown

```go
c := make(chan os.Signal, 1)
signal.Notify(c, os.Interrupt, syscall.SIGTERM)
<-c
client.Disconnect()   // closes socket; does NOT log out
```

`Disconnect()` is reversible — call `Connect()` again to resume with the same device. To fully unlink, use `client.Logout(ctx)`, which wipes the device.

## Logger

```go
log := waLog.Stdout("MyApp", "INFO", true /* color */)
// Levels accepted: "DEBUG" | "INFO" | "WARN" | "ERROR" | ""
// Sub-loggers: log.Sub("module")
```

`waLog.Logger` interface methods: `Warnf, Errorf, Infof, Debugf, Sub(module) Logger`.
