# FAIM's patches to http2

This directory is the `http2` crate's 0.5.20 release from crates.io, byte for byte
(`http2-0.5.20.crate`, sha256
`92d3114be2f413b2e491e686b93a28cda30c355cffc8d091a57f8be4b1342896`), with the changes
below. The workspace's `[patch.crates-io]` puts it in place of the release, and
`[workspace] exclude` keeps it out of the workspace's lints. FAIM's upstream leg runs
on it to send a client's own HTTP/2 bytes (ADR-0011).

Each change is its own commit after the import, so `git log -- vendor/http2` lists them.

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

FAIM's test of it is in `crates/faim-daemon/tests/integration/emulation.rs`: a stream
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
