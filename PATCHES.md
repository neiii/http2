# FAIM's patches to http2

This is FAIM's fork of the `http2` crate (0x676e67/http2, the HTTP/2 stack `wreq` uses).
Branch `faim` is the `v0.5.20` tag, whose `src/` matches the 0.5.20 release on crates.io
(`http2-0.5.20.crate`, sha256
`92d3114be2f413b2e491e686b93a28cda30c355cffc8d091a57f8be4b1342896`), plus the changes
below. faim-http's `[patch.crates-io]` pins a commit of this branch in place of the
release. FAIM's upstream leg runs on it to send a client's own HTTP/2 bytes (ADR-0011).

Each change is its own commit after the tag, so `git log v0.5.20..faim` lists them, and a
change to this branch gets a person's review, as faim-http's REVIEW.md asks of the pin.

## A GOAWAY refuses only the streams above the id it names (#451)

`src/proto/streams/streams.rs`, `src/proto/streams/send.rs`, `src/proto/connection.rs`.

The release treats any `GOAWAY` from the server as a connection error: `recv_go_away`
sets `conn_error`, so every later `send_request` and `poll_ready` fails, and once the
connection is idle it sends a `GOAWAY` of its own and closes. RFC 9113 §6.8 has a server
shut down gracefully by first naming the highest stream id, which refuses no stream,
then a lower one once it has seen what the client had in flight. A client may open
streams between the two, and curl does, so FAIM must be able to send them.

- `recv_go_away` keeps the server's error in `Actions::go_away` rather than
  `conn_error`. `send_request` and `poll_pending_open` refuse, with that error, only a
  stream whose id is above the last id the server's latest `GOAWAY` named
  (`Send::go_away_allows`).
- The connection closes on idle after a `GOAWAY` only when it names an id below the
  highest.

FAIM's test of it is in faim-http's `crates/faim-daemon/tests/integration/emulation.rs`: a stream
opened between a server's two `GOAWAY`s is answered over the same connection. The
release's own test suite isn't part of the published crate, so it doesn't run here.

## A client connection can move its next stream id up before its first stream (#473)

`src/client.rs`, `src/proto/streams/streams.rs`, `src/proto/streams/send.rs`,
`src/codec/error.rs`.

The release fixes a client connection's first stream id at the handshake
(`Builder::initial_stream_id`). FAIM starts the connection once it has read a client's
opening, and a client that sends its first request later, after FAIM's `SETTINGS`, may
open it on a stream other than the one FAIM guessed: Firefox opens on 3. FAIM must send
it on the client's id.

- `SendRequest::set_next_stream_id`, behind the `unstable` feature like
  `Builder::initial_stream_id`, sets the id the next stream takes, under the streams
  lock (`Send::set_next_stream_id`). The streams after it take the ids after it.
- It refuses, with the new `UserError::NextStreamIdNotAllowed` and the numbering left
  as it was, once a stream has taken an id (`Send` records that in `opened_stream`), and
  an id below the next one or of the other peer's parity.

FAIM's tests of it are in `crates/faim-proxy/src/h2_leg.rs` (the setter and each
refusal) and `crates/faim-daemon/tests/integration/emulation/first_stream.rs` (a
client's first request, sent late on stream 3, reaches the server on 3). The release's own
test suite isn't part of the published crate, so it doesn't run here.

## A stream's receive window grows without a WINDOW_UPDATE of its own (#490)

`src/client.rs`, `src/proto/streams/streams.rs`, `src/proto/streams/flow_control.rs`,
`src/proto/streams/mod.rs`, `src/proto/mod.rs`.

curl and Firefox raise each stream's receive window right after its `HEADERS`, with a
`WINDOW_UPDATE` of their own. FAIM sends the server those frames as the client wrote
them, so the stack on FAIM's side has to accept what the server may then send. The
release grows a stream's window only by the `WINDOW_UPDATE`s it writes itself, and only
after the application releases data. curl gives its first stream more than its later
ones (two `+0x9f0001` updates, then two `+0x9f0000` on each stream after), so no one
`initial_window_size` matches every stream.

- `SendRequest::recv_windows` hands out a `RecvWindows`, which holds the connection's
  stream state weakly: unlike a `SendRequest` clone, it doesn't keep the connection open.
  FAIM's frame rewriter below the connection holds one.
- `RecvWindows::grow` grows one stream's receive window, and the capacity the
  connection may announce on it, by an increment, writing no frame
  (`FlowControl::grow_window`). It refuses, changing neither, when the stream or the
  connection is gone or the window would pass 2^31 - 1.

FAIM's tests of it are in `crates/faim-daemon/tests/integration/emulation/stream_windows.rs`:
a body larger than the client's initial window, but within it once the client's update
is counted, arrives whole, where the release would refuse it with `FLOW_CONTROL_ERROR`.
