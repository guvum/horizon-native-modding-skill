# Joining servers of older releases (ViaVersion-like adapter)

Horizon speaks 26.3's protocol (777). `crates/horizon-protocol/src/adapter/` translates other
releases a step at a time, renumbering packet ids per state and registry ids inside packets
(`ids.rs`, generated `v*_ids.rs` tables) and rewriting packets whose layout changed
(`translate.rs`). Supported on 26.3 Nightlies (Nov 2026): 26.2, 26.1, **1.21.11 (774)**,
1.21.9/1.21.10 (773). A community draft extends it to 1.20.2–1.21.8 (PR #8718) — check open PRs
before duplicating.

## Switches

- Off by default: `horizon_client_lib::net::older_servers_enabled()` reads `HORIZON_OLDER_SERVERS=1`.
- Per connection, `net::older_server(address)` pings the server status and maps its protocol
  number with `ServerVersion::from_protocol`; `None` = speak 26.3.
- `ServerVersion::ALL` (newest first), `.name()` ("26.3", "1.21.11"…), `.protocol()`.

## A mod instead of an environment variable (pinned)

Cancellable HEADs on both functions plus a button on the Multiplayer screen:

```rust
// enabled.rs — target horizon_client_lib::net::older_servers_enabled, bindings = []
fn on() -> std::ops::ControlFlow<bool> { std::ops::ControlFlow::Break(true) }

// server.rs — target horizon_client_lib::net::older_server
fn pick(address: &str) -> std::ops::ControlFlow<Option<horizon_protocol::adapter::ServerVersion>> {
    use horizon_protocol::adapter::ServerVersion;
    match crate::net::my_state::choice() {          // a [[module]] in net.rs holding an AtomicU8
        None => std::ops::ControlFlow::Continue(()), // Auto: Horizon's ping
        Some(v) => std::ops::ControlFlow::Break(Some(v).filter(|v| *v != ServerVersion::Current)),
    }
}
```

UI: `screen_render_post` + `screen_mouse_button_pressed_pre` filtered on `screen.id == "multiplayer"`,
draw with `native_fill`/`native_text`, hit-test `screen.mouse`, store with `native_set_setting`,
load once in the `frame` hook. Left click → next version, right click → previous.

## Kicks on real 1.21.11 servers (66f3981, Nov 2026)

- **"Invalid signature for profile public key"**: Horizon sends the chat-session (profile key)
  packet from `horizon_client::game::remote_chat::Game::update_chat_session`, for 26.3 servers and
  for the adapter's older-release connections alike. A NoChatReports-style mod fixes it with a
  cancellable HEAD on that function (`ControlFlow::Break(())` while enabled). It's an internal
  target, so pin it. Verified joining a 1.21.11 Paper server behind Velocity: no kick.
- **Don't combine with another protocol adapter mod** (e.g. `viahorizon`): both rewrite the same
  connection. With `viahorizon 0.8.5-dev` the login stalled at the server's login-finished packet
  and dropped with "Timed out" after 30 s (reproduced offline by replaying captured Velocity login
  bytes); the Older Servers mod alone completed the login.
- **"You are already connected to this proxy!"**: the same account is already online, usually from
  the Java launcher. Not a Horizon bug; close the other client before testing.

## Item component layouts that differ (1.21.11 → 26.3)

Read in `horizon_protocol::item::older_layout(kind, r)` per quirk flag (`adapter::ids::quirks()`):

- **EitherHolder** (a bool, then the holder — or a `ResourceKey` string when false) in 1.21.11 for
  `jukebox_playable` (66), `instrument` (63), `provides_trim_material` (64), `damage_type` (8),
  `chicken/variant` (104), `zombie_nautilus/variant` (106) (26.3 ids). 26.1 made them plain holders.
  Symptom when missing: goat horn → `could not translate 1.21.11 packet 20 (Play): … 1 extra bytes`
  and `packet 101 … needed 1 bytes`. Fix: read the bool, holder → `Older::Pinned`, key → read the
  string, `Older::Dropped`. As a pinned mod:

  ```rust
  fn either(kind: i32, r: &mut Reader) -> std::ops::ControlFlow<Result<Older>> {
      use std::ops::ControlFlow::{Break, Continue};
      if !matches!(kind, 8 | 63 | 64 | 104 | 106)
          || !crate::adapter::ids::quirks().either_holder_song_and_tag_resistance { return Continue(()); }
      let read = |r: &mut Reader| -> Result<Older> {
          if r.bool()? { Ok(Older::Pinned) } else { r.string(32767)?; Ok(Older::Dropped) }
      };
      Break(read(r))
  }
  ```
- `damage_resistant` is a bare tag id (26.3: HolderSet), `use_remainder`/`container` hold ItemStacks,
  `pot_decorations` are item ids, dropped components (`map_color`, `swing_animation`) must be read
  past — upstream #8659 handles these; Nightlies before it break shulker boxes with contents, maps,
  decorated pots.

## Finding such bugs

Run a Paper server of the target version, join with the adapter on, fire commands over RCON one at
a time and diff the client log for `Horizon: could not translate <version> packet <id>` lines
(`scripts/bisect.py`). Map the pinned id to a name in `horizon-protocol/src/ids.rs`. Verify a
codec difference with `javap -c -p` on the old (Mojang-mapped Paper jar under
`versions/<ver>/paper-<ver>.jar`) and new (`…/minecraft/versions/26.3/26.3.jar`, not obfuscated)
classes, e.g. look for `EitherHolder.streamCodec` in `DataComponents`' static initializer.
