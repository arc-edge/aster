# Arc low-bandwidth mesh changes

This fork of [Aster](https://github.com/defenseunicorns/aster) carries the
changes [arc-uas](https://github.com/arc-edge/arc-uas) needs to run a small
shared ledger between drones over a shared radio, satcom or metered cellular.
Each one is opt-in, so a node that uses none of them behaves as upstream does,
except for three that reach every node: finite-TTL custody is on for any macOS node
with a boot domain, a node whose endpoint stops fails instead of spinning, and a
custody send charges a fixed second of age rather than 32 ms. All nodes of a mission
must set the same options.

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
  for a longer one, as does a peer whose last contact failed at the carrier when there
  is no gate. The higher identity of a pair waits one tick more, so while their waits
  end together the lower one goes first. After an outage both are past their waits and
  launch at once, and a slow link can hold up the lower one's connection past the tick.
  The higher identity then yields and its news goes in the inbound contact. If that
  contact fails, the news waits for the next publish or the idle period. Nothing records
  which peers had news due, so a node starts as if all of them did. A restart counts as
  news, at the cost of one contact per peer in reach, and of retries if that contact
  fails. Pacing also turns off path MTU discovery: its probes are the largest single
  cost of a connection this short, and each contact is a new connection.
- `ContactGate`, through `with_contact_gate`. A contact with a peer out of range
  still spends airtime, because the handshake is sent and retried until it times
  out. On a shared radio with ten nodes that cost more than everything else. An
  application that knows who it can hear keeps this current, and outbound contacts
  go only to those peers. With a gate a contact that fails gives back the news it
  took and doesn't mark the peer unreachable, so the retry waits only the news spacing
  and news goes as soon as the peer returns. There is no back-off: a peer with news due
  is retried at the news spacing until a contact succeeds or the gate closes, and each
  retry costs at least a handshake. Only a peer with no news due stays on the idle
  period. A contact that runs out its 30 s deadline doesn't give its news back. On a link
  too slow for that page a retry fails the same way, and the lower identity's retries
  hold off the higher identity's own contact, so the peer waits for the next publish or
  the idle period. The cost is that a backlog too big for one contact moves on only then.
- `MutableSourceInterests::event_only`. Contacts end after Event transfer and
  skip the State, Record, Blob and Event-bridge lanes, which cost round trips on
  every contact even with nothing to move.

  That leaves a contact 30 s in all, because only Blob transfer counts as progress, and
  a page of up to 16 finite Events lands whole. So a link has a floor. Measured with
  300 ms each way, a backlog of 24 Events cleared at 8 kB/s and none of it landed at
  6 kB/s, and at 2 kB/s a backlog of 2 landed and one of 6 didn't. Below about 8 kB/s a
  backlog stays stuck until enough of it expires.
- `MutableSourceInterests::with_min_send_remaining`. A contact's schedule is exact,
  so one Event that expires before its page is written aborts the whole contact.
  After an outage a backlog always holds Events at the end of their lifetime, and
  on a slow link every retry aborted the same way, so the link never recovered.
  With this set, an Event with too little life left isn't scheduled. It stays in
  the difference as unscheduled, which the protocol already allows.

  The same option decides what a receiver reconciles against. An Event that expires
  leaves a fence that is never removed, and with every fence in the baseline the
  sender and receiver sets stopped matching, so negentropy listed ids on every
  range: a byte model gave 960 kB per contact in the second hour of a mission against
  1.3 kB with no fences. No peer schedules an Event that close to expiry, so with this
  set its fence stays out. A late offer of an accepted Event under Consume lands as a
  duplicate. An offer fails the contact on an accepted Event under Carry and on a
  route-only one under Consume, so those fences stay in. A route-only fence under Carry
  stays in too, though an offer of it lands as a duplicate. So do quota fences, because
  peers still hold those Events. Every fence counts against a store's lifetime limit
  of 262,144 Events, live rows included. Past it a node refuses new Events, and a
  contact carrying one fails.

  A reboot changes the custody clock, so a finite row loses its age for good and is
  withheld until its lifetime ends. It sat in neither set, so every peer sent it again
  on every contact, and a duplicate can't clear the loss. With this set such a row goes
  in the baseline as a fence. With 100 of them a contact measured 9.7 kB rather than
  1.1 MB, using inline credentials. A route-only row under Consume stays out, because
  it still wants its content.

  The fences have a cost, because no sender can offer a row a node has lost. A rebooted
  node's own sender set lacks these rows, so its lane to each peer lists them. A peer's
  lane to it lists them too once the peer's copies are gone, and both lanes do when
  both ends rebooted. That is about 35 bytes per row per lane on every contact, for up
  to a TTL after boot, since time spent down doesn't count toward a row's age. One
  re-send of a row costs as much as about 110 contacts of listing it, so fencing still
  wins while peers hold rows.
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
- `CarrierError::EndpointStopped`. When a bound address leaves its interface the Iroh
  endpoint can stop for good, and accept then returns at once forever. The accept loop
  spun on that while application commands kept succeeding, so nothing restarted the
  service. A stopped endpoint now ends the node: commands fail, and
  `RunningNode::shutdown` says why, without waiting to close the endpoint, which might
  never finish. Either network stopping does this. A restart fails until the main
  address is back, and comes up without an extra one that is gone, so a main address
  leaving takes the extra network down with it until that address returns, though an
  application can start the node with the other address as main until then. A failed
  handshake from one peer stays an ordinary contact error. This reaches every node,
  with no option to turn it off.
- Finite-TTL custody on macOS. `clock_gettime(3)` documents `CLOCK_MONOTONIC` as
  continuing through sleep, which is the property finite custody needs, and the
  custody clock already read it. `kern.bootsessionuuid` is the boot domain, because
  `kern.boottime` changes with the time zone and the wall clock, so a restart would
  read as a reboot. This reaches every macOS node where that sysctl can be read, with
  no option to turn it off.
- The custody finalization charge. A custody send adds a fixed age to an Event for
  the time sealing its frame takes, and aborts the whole contact if sealing took longer.
  The check is page-wide, so the first entry's charge has to cover the whole page.
  Upstream's 32 ms aborted contacts on a quiet machine in a 10-node emulation, and
  can't survive normal scheduling on a Jetson or a busy laptop, so it is now one second.
  Each hop adds about a second to an Event's age, which is small against a lifetime of
  minutes, and no receiver limits what a hop adds. An Event with about a second or less
  to live can't be sent, and one scheduled anyway aborts its contact, so keep
  `with_min_send_remaining` above that. This reaches every node, with no option to turn
  it off.

## Keeping up with upstream

Work happens on `arc/low-bandwidth-mesh`, and arc-uas pins a revision of it.
Rebase onto upstream `main` to take new work. Aster is pre-release and its store
format has no migration promise, so every node of a mission must run one build.
