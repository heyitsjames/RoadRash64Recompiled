# Full 3-4 player HUD

**Graphics → Full 3-4 Player HUD** (default off) shows the original two-player
health bars and weapon box in every view of 3- and 4-player split screen.

## Native behavior

The HUD producer `80030220` draws health and weapons only for layout 0
(single/online, `31D5C`) and layout 1 (two-player, `323F0..33234`). For layouts
of 2 or more, `323E8` branches to `33238`, skipping that block; the quadrant
path that follows draws only rank/status and speed.

The two-player block's upper panel, in 320x240 units:

- rider bar `x24..60, y17..21`, fill `rider+310 / rider+30C`, red to green over 200;
- bike bar `x24..60, y28..33`, fill `bike+4F8 / bike+4FC`, red to green over 255;
- weapon box sprite `3D` at `(40,25)`, icon sprite `A3+weapon` at `(72,25)`;
- quantity label (`%d`, half scale) beside the icon when two or more are held;
- pickup indicator (sprite table `8009F3DC`) at half scale, steady while its
  timer is above 5, otherwise blinking.

## Implementation

`rr64_quadrant_hud_draw` runs at `30220`'s shared exit (`342C4`), after the
display-list cursor has been stored. For each active view it reproduces that
panel with the same assets and producers the stock block uses: the bar render
state, prim-colour and texture-rectangle commands are appended to the display
list, and the box, icon, indicator and label are queued through the native
sprite (`8001EB90`) and label (`800795AC`) producers. No game code is re-entered
and no layout global is changed.

Each panel is drawn at about 60% of its two-player size in its view's bottom
left, mirroring the native quadrant speedometer at the bottom right (8 units
from the view edge, centred 105 units down). Rank/status stays top left.

RT64 in this project anchors unannotated rectangles in a view's upper-left or
lower-left region to the screen edge (its automatic race-HUD placement). Applied
per draw, that would split the panel: the box would move and the icon would
not. Every panel draw therefore carries an explicit centre `SetRectAlign`,
which keeps its authored 4:3 position and exempts it from that heuristic. The
bars are wrapped directly in the display list; the queued box, icon and
indicator are wrapped by hooks around each record in the sprite flush
(`8001EF8C` at `1F19C`/`1F5CC`); the quantity label is pinned through
`rr64_hud_label_pin_center`.

Finished or busted riders (stock flag `sp+68+view*2`) get no panel, matching
the stock HUD. Paused frames, a cleared HUD mask and a nearly full display list
or sprite pool are left untouched.

## Verification

`RR64QuadrantHudSmoke` checks gating, per-view bar geometry and colour, weapon
box/icon/label/indicator calls and placement, explicit alignment of bars and
queued panel sprites, label pinning, finished-view skipping,
restoration of the caller's argument area and the capacity guard. These are
offline checks against a fixture; on-screen appearance needs a visual check.

The two-player distance-to-finish counter is not included here.
