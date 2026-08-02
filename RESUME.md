# Bike Racing Game, current project state

Updated 2026-08-01. Everything in this file has been checked against the actual running
build. Read this whole file first, then go to "Next actions" at the bottom.

Standing rule: aim for the highest possible quality, cost is not a limiting factor.
"Working" is the minimum bar, not the goal.

---

## Where things stand right now

TypeScript compiles with 0 errors. The GLSL check script passes cleanly. The game renders
using 49 shader programs with no graphics errors. There are 13 commits so far, the most
recent one being a recovery of work from an interrupted multi-agent editing session.

```bash
npm run dev                                   # http://127.0.0.1:5173
node tools/capture/capture.mjs --poses        # 16 high-res screenshots
node tools/capture/capture.mjs --seq all      # 8 sequences plus contact sheets
npm run check:glsl                            # checks for a specific bug, also runs during build
```

Eight automated agents were working on this project in parallel and got cut off partway
through due to a session limit. Their unfinished work was kept anyway, since it still
compiles, passes the check, and renders correctly. Two things needed fixing afterward: two
stray backtick characters inside GLSL code comments (the check script caught these), and a
tire configuration setting that had been added but not filled in with actual values, which
was completed using the documented formula (front trail = wheel radius divided by tangent
of head angle, giving 0.0766 meters).

Each of the eight agents had been told to first look at their own code changes and re-test
visually before making further edits. Anything they didn't get to finish is still an open
issue, listed in the defects section below.

---

## THE TESTING TOOL ITSELF HAS CAUSED MOST OF THE FALSE ALARMS

Five separate times, someone reviewing the game reported a bug that turned out to actually
be a problem with the testing/screenshot tool, not the game itself. Always check the
testing tool first when something looks impossible.

1. **The warm-up period was eating the actual jump.** The bike leaves the ground 127
   physics steps after the warm-up period ends, but a screenshot was being taken only 24
   steps after warm-up ended, so it never captured any airtime. Three separate review
   passes looked at footage of a jump that had already happened off-screen before
   recording started.
2. **Five out of eight test sequences accidentally recorded the exact same run twice**
   (frames were nearly pixel-identical). They looked the same because the character never
   actually left the ground, so a supposedly different input made no visible difference.
3. **The screenshot tool captures all 16 test poses on a single page without resetting
   effects between them**, so a visual effect from one screenshot was leaking into the
   next one. A motion blur effect got stuck partway on during a "crash" screenshot and
   then bled into a close-up screenshot that should have had zero blur. This has been
   fixed by resetting all effects between screenshots, and any new effect added later must
   also be included in that reset.
4. **Contact sheets (grids of frames for review) were shrinking images down 3.33x** before
   review, so a rider that measured 150 pixels tall on the shrunk contact sheet was
   actually 254 pixels tall in the real frame. Both measurements were technically correct,
   they were just measuring different-sized versions of the same image. Fixed by making
   the contact sheet grid bigger.
5. **Rival racers were spawning only 3.5 meters behind the player, but the chase camera
   sits 4.8 meters back**, meaning the camera was accidentally spawning right where a
   rival racer was placed. Moving them further back to 12 meters just caused a different
   problem, a screenshot meant to show a "pack of racers" now showed no rivals at all in
   frame. Fixed by having rivals for that specific screenshot spawn ahead of the player
   instead of behind.

Also fixed: the in-game timer was showing 0:00.19 even though the run was 96% finished,
because teleporting the character during testing never advanced the game clock. The timer
now calculates time based on distance traveled instead.

---

## THE BUGS THAT ACTUALLY MATTERED, AND WHAT EACH ONE TEACHES

None of these were caught by TypeScript, and none were obvious just from running the game.
Every one was only found by actually measuring something specific.

1. **The rendering loop was feeding in the wrong kind of number for time.** Instead of
   getting "time passed since last frame" (a tiny fraction of a second), the whole visual
   side of the game was advancing a full one second per rendered frame. Three different
   agents discovered this independently. As a result, dust particles were being created
   and destroyed before they ever got drawn on screen even once.
2. **A comparison that was supposed to find the highest value among nearby points was
   using strict "greater than," but two points 3.5 meters apart both hit the maximum value
   of exactly 1.0.** Since it was strict "greater than" rather than "greater than or equal
   to," whichever point came first in the loop won by default, causing every point on the
   ground to incorrectly copy its height from up to 5 meters further back up the track.
3. **A jump ramp was accidentally getting its height added twice at one specific point**,
   because two separate loops both included that same point. This created a a very steep
   4.2 meter rise over just half a meter of horizontal distance, an 83 degree angle, which
   then caused the terrain's center-line to fold back on itself when resampled.
4. **A texture that was supposed to tile seamlessly never actually tiled**, because the
   noise pattern's frequency didn't evenly divide into the tile size. Every texture that
   was supposed to repeat seamlessly instead had a visible seam.
5. **A visible seam in the sky was caused by how the sky texture's detail level was chosen
   based on the rate of change of the viewing direction, not the viewing direction itself.**
   Since the sky is a dome made of triangles, a straight line running down the middle of
   the dome (a meridian) creates a sudden jump in that rate of change, which caused the
   graphics card to pick a different texture detail level right along that line.
6. **Distance fog was supposed to be applied in fixed steps/bands, but the code was still
   blending it smoothly, and on top of that, the boundaries between fog bands were placed
   between 560 and 1855 meters away while the actual camera shots only cover 200 to 900
   meters.** Fixing only the blending issue wouldn't have changed anything, since the fog
   bands weren't even in the right range to matter.
7. **The bike's resistance to tipping over sideways was almost zero**, because the force
   trying to keep it upright and the force of gravity pulling it over were almost exactly
   canceling each other out. This meant the bike's lean angle had nothing actually stopping
   it once it started tipping.
8. **In test screenshots, the bike's starting position was set exactly at ground level**,
   which buried both wheels 0.27 meters underground. This meant the suspension system was
   pushing back with an unrealistically large force, no wheel ever properly touched the
   ground, so no grip was calculated and no dust effects appeared.
9. **A fog setting that controls haze strength very close to the camera was set to 0.16
   instead of near zero, adding a layer of haze right at the camera lens itself.** This one
   bug survived four separate rounds of review. Turning fog off versus on changed the
   overall color noticeably, when it was supposed to barely change it at all.
10. **Rider name labels and rider color indicators were being pulled from two completely
    separate lists that didn't match up.** This caused the leaderboard to show one rider's
    name next to a different rider's color, every single frame.

### Standing rules going forward
- **Never put a backtick character inside a GLSL code comment.** This mistake has already
  cost three full rounds of fixing. There's now an automated check for it
  (`npm run check:glsl`). Worth noting: the very first version of that check had the exact
  same bug it was meant to catch, it treated the first backtick it found as the end of the
  comment, which is wrong, and so it agreed with the broken code and reported everything as
  fine. It's been rewritten to scan line by line instead, tested against six known cases,
  and it correctly caught a real bug the first time it was used for real.
- The specific 3D graphics setup being used (Three.js with GLSL3) does not automatically
  provide certain output variables that older setups do, code must explicitly account for
  this.
- **Automated reviewers are good at spotting that something is wrong, but unreliable at
  correctly explaining why.** In several cases, an agent investigating a bug found that
  its original theory was wrong: a "sky seam" turned out to not be caused by the sun
  shafts, a camera issue was actually just another racer blocking the view, a jagged trail
  edge was an ink/outline problem rather than a geometry problem, and a suspected terrain
  detection bug was actually fine, the real issue was two unrelated multipliers further
  down the pipeline canceling it out.
- **Automated agents test things in isolation and can be wrong about how the finished game
  actually behaves.** Always double check using the real screenshot/recording tool.

---

## OPEN ISSUES (as of review round 4, still not fully passing)

These are what the eight interrupted agents were working on. Each one should be
re-checked before doing more work, since some may have been partially fixed already by
salvaged work.

**Rider character** (`src/rider/*`) — feet are never actually on the pedals in any frame
(both feet appear at the bottom of the pedal stroke at once, which is physically
impossible on a normal bike); the rider's body doesn't lean and turn together with the
bike (bike leans 40 degrees, rider's spine barely moves); no visible pedaling animation;
legs appear to end mid-shin from certain camera angles; a hand comes off the handlebars
outside of a crash; the back of the helmet renders looking like a face when viewed up
close; crashes are just a quick 167 millisecond pose swap, and both riders play the exact
same mirrored crash animation.

**Physics** (`src/bike/*`) — holding full steering lock for 2 seconds produces zero change
in the direction the bike is actually facing; leaning is purely visual and disconnected
from actual turning. Downhill coasting deceleration and acceleration numbers look
unrealistic. The crash recovery speed increases oddly (1 to 7 km/h). **The gap jump on the
course cannot actually be cleared** — the rider launches early off a natural bump before
the real jump, doesn't get enough height, lands past the far edge, and ends up falling
through and accelerating to 150 km/h inside the mountain terrain. A similar jump elsewhere
on the course was already fixed the same way and now gives proper height, the gap jump
needs the same kind of fix. One of the interrupted agents had already started adding the
physics terms needed to connect leaning and turning together, which is the right direction
for fixing the steering issue.

**Camera** (`src/fx/CameraDirector.ts`) — the amount the picture changes frame to frame
stays flat even as speed increases by 19%, meaning the camera doesn't actually feel faster
as the bike goes faster. No camera whip, field-of-view kick, swing-around, or slow-motion
effect shows up in any test sequence. The camera during switchback turns is now too tight
and clips the rider out of frame at the bottom. The crash camera sequence loses the player
out of frame entirely partway through.

**Effects** (`src/fx/DustSystem.ts` etc.) — dust appears about 1.5 meters above where the
wheel actually touches the ground, never fades away, and doesn't appear at all during a
crash impact. The impact flash effect fires 267 to 433 milliseconds after the actual
impact instead of right when it happens, and only brightens half the screen. (The wheel
motion blur effect has since been fixed, it now correctly kicks in at higher speeds.)

**Terrain** (`src/terrain/TerrainMaterial.ts`, `Zones.ts`) — a previous fix for color
banding only worked at the exact spot it was tested on and does not hold up elsewhere on
the course. Needs to be re-verified across multiple locations and camera angles, not just
the one that was originally tested. Water still looks too grey/dull compared to the
intended color. There's also leftover unwanted snow speckling in one section of the
course.

**Scattered objects** (`src/terrain/Scatter.ts`) — boulders along the stream section are
rendering as broken, disconnected triangles, with outline strokes going off into empty
space and solid colored shapes floating incorrectly. Rocks elsewhere on the course render
as wireframe outlines instead of solid shapes. Both problems likely share one root cause,
a step needed to properly smooth outlines isn't running on these specific objects.

**Outline/line rendering** (`src/npr/passes/LinesPass.ts`) — stray outline strokes and
closed shapes are appearing on flat ground where they shouldn't be (terrain chunk
boundaries and internal texture boundaries are incorrectly being outlined as if they were
real edges); unwanted concentric circles appear on the rider's helmet.

**Sky** (`src/npr/Sky.ts`) — the sun shaft/glow effect creates the single largest smooth
color gradient anywhere in the game, when everything else uses hard-edged bands on
purpose. There's also a large solid orange rectangle floating incorrectly above the
horizon, and small unwanted cloud fragments scattered in one part of the sky.

**HUD (on-screen display)** — a progress marker visually overlaps itself at the 76% mark.
Nobody has been assigned to fix this yet.

**Closest to being finished:** the bike detail view and the ridge exposure view.
**Confirmed fixed, don't touch again:** ridgeline outline ink, distance display, motion
blur effect resetting properly, cloud shapes, snow speckle (in the areas already checked),
terrain path carving, track edge geometry, sky seams running top to bottom, HUD text
overlap, corner preview indicator.

---

## NOT STARTED YET

- **Performance optimization pass.** Target is a locked 60 frames per second at high
  resolution on a fast modern machine. Only one measurement has been taken so far: the
  main rendering pass alone takes 22 milliseconds (187 objects casting shadows, 195 objects
  in the prepass), compared to 0.025 milliseconds for outline rendering and 3.77
  milliseconds for shadows. That puts current performance at around 45 frames per second,
  and the main rendering pass is clearly the bottleneck, not the visual effects. Shadow
  memory usage is around 168 MB after a recent quality increase.
- **Audio has not been listened to or tested at all yet.**
- **The replay system and "biggest air" cinematic camera are built but have never actually
  been tested.**

---

## Next steps, in order

1. Restart the eight parallel agents on their remaining work (each one needs to be told
   exactly where their predecessor left off, and told to review their own changes before
   editing anything further).
2. Run a full fresh review pass using both reviewers.
3. Fix the HUD progress marker overlap, currently unassigned.
4. **The performance optimization pass.** This has never actually been measured in a real
   play session, and the main rendering pass is already over the performance budget.
