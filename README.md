# FrameGuard

**English** · [Русский](README-RU.md)

A Valheim mod (planned): it adjusts graphics to the load by itself. When the frame rate drops
below the target because of rendering — a heavy area, a big base, a fight full of effects — it
lowers what shows least for a while, and brings it back once there is room. When the game's
logic is what is slow (AI, physics, network), it leaves the graphics alone.

**Status: plan only, no code yet.** The implementation plan (in Russian) is in
[PLAN.md](PLAN.md).

What it will do differently from what exists:

- finds the bottleneck from the engine's frame steps (no GPU timings needed — they are off in
  Valheim's release build);
- a separate ladder of settings for a GPU limit and for a CPU rendering limit, from least to
  most visible: shadow distance, render scale, shadow resolution, draw distance;
- checks every step and undoes one that brought nothing;
- ignores hitches (loading, autosave, GC) and pauses on teleports, menus and the map;
- smooth transitions; lowers quickly, raises slowly and only with measured headroom;
- your graphics settings are the ceiling, and nothing is written to the game's saved settings.

Client-side only. License: MIT.

*Planned with the help of an AI assistant (Claude by Anthropic).*
