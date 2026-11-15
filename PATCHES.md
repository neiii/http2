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
