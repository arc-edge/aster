# Arc low-bandwidth mesh changes

This fork of [Aster](https://github.com/defenseunicorns/aster) carries the
changes [arc-uas](https://github.com/arc-edge/arc-uas) needs to run a small
shared ledger between drones over a shared radio, satcom or metered cellular.
Each one is opt-in, so a node that uses none of them behaves as upstream does.
All nodes of a mission must set the same options.

Measured on one link between two nodes, with one 10-byte Event:

| | Upstream | This fork |
| --- | --- | --- |
| Bytes per Event in a page | 10,533 | 3,858 |
| Handshake bytes per contact | 22,882 | 9,596 |
| Protected frames per contact | 84 | 32 |
| Round trips per contact | 14 | 6 |
| Load on an idle link | 65 kB/s | under 2 kB/s |

## What changed

- `SelectedForwardingConfig::with_contact_pacing`. A new Event starts a contact
  at once upstream, so a steady publisher pays a full contact per Event. Pacing
  batches news behind a minimum interval, and a link with no news stays quiet
  for a longer one, as does a peer whose last contact failed. The higher identity
  of a pair waits one tick more, so the two sides don't collide. Pacing also turns
  off path MTU discovery: its probes are the largest single cost of a connection
  this short, and each contact is a new connection.
- `ContactGate`, through `with_contact_gate`. A contact with a peer out of range
  still spends airtime, because the handshake is sent and retried until it times
  out. On a shared radio with ten nodes that cost more than everything else. An
  application that knows who it can hear keeps this current, and outbound contacts
  go only to those peers. With a gate a failed contact doesn't mark a peer unreachable,
  because the application already says who is in reach, so news goes as soon as a peer
  returns.
- `MutableSourceInterests::event_only`. Contacts end after Event transfer and
  skip the State, Record, Blob and Event-bridge lanes, which cost round trips on
  every contact even with nothing to move.
- `MutableSourceInterests::with_min_send_remaining`. A contact's schedule is exact,
  so one Event that expires before its page is written aborts the whole contact.
  After an outage a backlog always holds Events at the end of their lifetime, and
  on a slow link every retry aborted the same way, so the link never recovered.
  With this set, an Event with too little life left isn't scheduled. It stays in
  the difference as unscheduled, which the protocol already allows.
- `MissionCredentialTable`, through
  `UnprotectedReferenceMission::with_credential_table`. A source envelope and a
  handshake flight carry each member's credential and the authority's signature
  over it, about 6.6 kB that never changes during a mission. With a table every
  node holds, they are left out and looked up by id. The table is not trusted:
  entries go through the same credential, authority and publisher checks an
  inline credential does, a bad entry fails authentication, and an absent one
  fails closed.
- `ExtraBind`, through `SelectedForwardingConfig::with_extra_bind`. A second
  network gets its own Iroh endpoint with the same identity, and a peer's subnet
  picks the endpoint. Iroh offers an endpoint's addresses to peers and probes
  them on every connection, which can't be turned off, so two sockets on one
  endpoint put traffic on the second network whenever the first one works. Bind
  exact addresses rather than a wildcard for the same reason.
- `SelectedForwardingConfig::with_peer_fallbacks`. A second address for a
  configured peer, tried only after its own address fails to connect.
- Finite-TTL custody on macOS. `clock_gettime(3)` documents `CLOCK_MONOTONIC` as
  continuing through sleep, which is the property finite custody needs, and the
  custody clock already read it with `kern.boottime` as the boot domain.

## Keeping up with upstream

Work happens on `arc/low-bandwidth-mesh`, and arc-uas pins a revision of it.
Rebase onto upstream `main` to take new work. Aster is pre-release and its store
format has no migration promise, so every node of a mission must run one build.
