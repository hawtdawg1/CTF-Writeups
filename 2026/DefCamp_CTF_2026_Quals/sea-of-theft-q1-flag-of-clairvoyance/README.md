# sea-of-theft - Q1: Flag of Clairvoyance

**Category:** Misc / Web  
**Difficulty:** Hard  
**Flag:** `DCTF{W4llh4ck1N6_1s_s0_m3Ta}`

> Not all is as it seems.

## Challenge Overview

The game renders the flag on-screen every frame as giant, near-invisible wavy text painted onto a walled-off island. A reef damage ring, the "waves," prevents a normal player from reaching it. Since the client already draws the text, we can read it without entering the reef: hook the 2D canvas text function. That is the wallhack hinted at by "Not all is as it seems."

## Recon

[`https://sot-chall.abcjr.dev/`](https://sot-chall.abcjr.dev/) serves **PIPWORLD2**, a top-down multiplayer pirate `.io` game. Its client is written in Rust and compiled to WebAssembly with Trunk/wasm-bindgen. It talks a custom binary protocol to the game server over WebSocket (`wss://sot-chall-test-prod-server.abcjr.dev`). A manifest on a separate versions host pins the release.

The client artifacts include:

- `pipworld2-client-*_bg.wasm` and its JavaScript glue;
- `pipworld2.dat`, the balance/datapack; and
- asset packs.

Several strings in the WASM stood out:

- `FLAG:` - a format string used by a text-render path;
- `WALL` - a label drawn in the world;
- `world.reef_dps`, `tide.amplitude`, and `world.reef_ramp` - a damaging reef/tide; and
- `debug=1`, `debug=true`, `dev`, `show HUD`, and a dev-command-form console - a compiled-in developer overlay gated off in the production build.

Searching the running client's WASM linear memory for `DCTF{` found nothing. The flag is not a static string in the client; it arrives as game state and is drawn.

## Finding the flag text

Decompiling the render path showed that the flag is painted on the canvas in a world-space transform (scale about `0.4`) as a row of individual glyphs. The text is nearly invisible against the sea and sits on a `WALL` island surrounded by the reef. Although the player cannot safely reach the island, the client calls `CanvasRenderingContext2D.fillText` for the glyphs whenever they are on screen.

The client already has the information and relies on presentation to hide it. Reading the draw calls exposes it.

## Exploitation

I used headless Chromium through the Chrome DevTools Protocol. A pre-load script captured the WebSocket and WASM memory and wrapped the canvas text function.

### 1. Hook the text renderer

Wrap `CanvasRenderingContext2D.prototype.fillText`. For each call in world space (transform scale below `0.9`), record the glyph and its screen position, then repaint it opaque red:

```js
const proto = CanvasRenderingContext2D.prototype;
const orig = proto.fillText;
proto.fillText = function (t, x, y) {
  try {
    const m = this.getTransform();
    if (m.a < 0.9) {                     // world-space draw, not the HUD
      const e = m.a * x + m.c * y + m.e; // glyph screen X
      const f = m.b * x + m.d * y + m.f; // glyph screen Y
      window.__caught.rec.push([String(t), e, f]);
      this.save();
      this.globalAlpha = 1; this.fillStyle = '#ff2b2b';
      const r = orig.call(this, t, x, y);
      this.restore();
      return r;
    }
  } catch (_) {}
  return orig.apply(this, arguments);
};
```

### 2. Sail next to the wall

The flag glyphs render only while the `WALL` island is on screen. Steer the ship until the `WALL` label appears among the captured draw calls, then center the view on it. There is no need to enter the reef: on-screen text renders even when the island is unreachable, and the letters are on its near edge.

### 3. Stitch the row together

The flag row is wider than one screen. Use the fixed `WALL` label as an anchor and capture glyphs relative to it while panning left and right. Merge the overlapping windows into one string:

```text
DCTF{W4llh4ck1N6_1s_s0_m3Ta}
```

## Flag

```text
DCTF{W4llh4ck1N6_1s_s0_m3Ta}
```

"Wallhacking is so meta" - the flag itself is a wallhack.

## Notes

- Anything the client can draw, the player can read. Fog of war, low-contrast text, and unreachable areas are presentation rather than security. Server-side visibility culling is the real fix.
- Hooking `fillText`/`strokeText` (or, for WebGL games, the geometry/label upload path) is a useful way to see everything a client renders.
- This client also ships a production-gated developer console (`?debug=1`, `xp`, and other commands) and streams full world state to every player. Those facts help with the other two questions in the challenge.
