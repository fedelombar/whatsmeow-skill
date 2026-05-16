# Groups & contacts

## Group operations

All take `ctx` and a group JID (`g.us` server). Sources in `group.go`.

| Operation | Method |
|---|---|
| Create | `client.CreateGroup(ctx, whatsmeow.ReqCreateGroup{Name: "...", Participants: []types.JID{...}})` (`:56`) |
| Get one | `client.GetGroupInfo(ctx, groupJID)` → `*types.GroupInfo` (`:591`) |
| List joined | `client.GetJoinedGroups(ctx)` → `[]*types.GroupInfo` (`:503`) |
| Add/remove/promote/demote | `client.UpdateGroupParticipants(ctx, groupJID, jids, action)` (`:189`) |
| Rename | `client.SetGroupName(ctx, groupJID, "New name")` (`:324`) |
| Topic / description | `client.SetGroupTopic(ctx, groupJID, prevID, newID, topic)` (`:337`) |
| Photo | `client.SetGroupPhoto(ctx, groupJID, jpegBytes)` (`:292`) |
| Get invite link | `client.GetGroupInviteLink(ctx, groupJID, reset)` (`:393`) — pass `reset=true` to revoke old link |
| Look up from link | `client.GetGroupInfoFromLink(ctx, code)` (`:457`) — `code` is the part after `chat.whatsapp.com/` |
| Join via link | `client.JoinGroupWithLink(ctx, code)` → `types.JID` (`:478`) |
| Look up from invite IQ | `client.GetGroupInfoFromInvite(ctx, jid, inviter, code, expiration)` (`:418`) |

### `ParticipantChange` actions

`UpdateGroupParticipants`'s action arg is one of:

- `whatsmeow.ParticipantChangeAdd`
- `whatsmeow.ParticipantChangeRemove`
- `whatsmeow.ParticipantChangePromote`
- `whatsmeow.ParticipantChangeDemote`

### `ReqCreateGroup`

```go
client.CreateGroup(ctx, whatsmeow.ReqCreateGroup{
    Name:         "Project X",
    Participants: []types.JID{user1, user2},
    // GroupParent / GroupLinkedParent fields exist for community sub-groups.
})
```

Returns the new group's `*types.GroupInfo`.

## Contact store

Accessed as `client.Store.Contacts`, which implements `store.ContactStore` (`store/store.go:102-110`):

```go
type ContactStore interface {
    PutPushName(ctx, user, pushName) (changed bool, previousName string, err error)
    PutBusinessName(ctx, user, businessName) (changed bool, previousName string, err error)
    PutContactName(ctx, user, firstName, fullName) error
    PutAllContactNames(ctx, contacts) error
    GetContact(ctx, user) (ContactInfo, error)
    GetAllContacts(ctx) (map[JID]ContactInfo, error)
}
```

```go
info, err := client.Store.Contacts.GetContact(ctx, userJID)
// info.FullName, info.FirstName, info.PushName, info.BusinessName, info.Found
all, err := client.Store.Contacts.GetAllContacts(ctx)
```

Contact data is populated automatically as the library sees messages and presence updates; `HistorySync` events also seed the store with the full address book.

## User info (over the network)

For data NOT in the local store (privacy settings, status text, presence):

```go
info, err := client.GetUserInfo([]types.JID{userJID})       // batched
// info[userJID].Status, .PictureID, .Devices, .VerifiedName
```

## Business profiles

```go
profile, err := client.GetBusinessProfile(ctx, businessJID) // user.go:413
// profile.JID, .Address, .Email, .Categories, .ProfileOptions, .BusinessHours
```
