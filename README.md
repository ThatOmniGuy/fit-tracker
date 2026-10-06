# Fit Tracker

A personal, offline weight-loss and workout tracker. It's a single web page (`docs/index.html`) with no frameworks, no build step and no network calls. Everything you log stays on your device.

```
docs/                 ← the app (this folder is what gets published)
  index.html          the whole app: styles, plan data, logic, screens
  sw.js               offline cache (lets the iPhone app open with no signal)
  manifest.webmanifest, icon-180.png, icon-512.png
tests/run-tests.js    plan and logic tests (node tests/run-tests.js), kept on the PC, not uploaded
tools/make-icons.py   regenerates the icons
```

**Privacy:** no personal details are in any file. Your weights, goal and dates are entered in the app on first run and stored only in that device's browser storage. That's why it's safe to put this repo on a public GitHub account.

---

## On the PC

Double-click `docs/index.html`. It opens in your default browser (Chrome or Edge recommended) and works fully offline.

Data is saved in that browser, for that file location. If you move the file or switch browsers you'll see a fresh app. Use **Settings → Backup** to carry data over.

---

## On the iPhone: GitHub Pages + Add to Home Screen

### Why this route

- Opening an HTML file from the Files app on iOS doesn't reliably run JavaScript or keep storage, so that's out.
- Serving it from the PC over Wi-Fi (`python -m http.server`) works, but it's plain HTTP, so the offline cache can't run. The PC has to be on and serving every time you open the app. Saved data is also tied to the PC's IP address, so if the router hands out a new one, the app looks empty.
- A free HTTPS host (GitHub Pages) lets the service worker cache the app on the phone. After the first load it opens in airplane mode, and nothing you log is ever sent anywhere. The network is only used to download the app itself, once, plus updates when you choose.

### One-time setup (about 10 minutes)

1. Sign in to GitHub (a free personal account is fine) and create a **new public repository**, e.g. `fit-tracker`. Free accounts need the repo to be public for Pages. That's fine, because no personal details are in the files.
2. Upload this project: on the repo page choose **Add file → Upload files** and drag in the `docs` folder, `tests`, `tools` and `README.md`. Don't include the original brief or any backup `.json` files. Commit.
   - Or with git: `git init`, `git add .`, `git commit -m "Fit Tracker"`, then push to the new repo.
3. In the repo go to **Settings → Pages**. Under *Build and deployment* pick **Deploy from a branch**, branch **main**, folder **/docs**, then Save.
4. Wait a minute or two. The page shows your address, like `https://<your-username>.github.io/fit-tracker/`.
5. On the iPhone, open that address in **Safari**. Other browsers can't add proper Home Screen apps on older iOS versions.
6. Tap **Share → Add to Home Screen → Add**.
7. **Open the app from the new Home Screen icon** and do the first-run setup there. The Home Screen app has its own storage, separate from the Safari tab, and it's the one iOS won't clear for inactivity.
8. To test offline: close the app, turn on airplane mode, and open it again. It should load normally.

### Honest notes about iOS storage

- Safari can delete a website's storage after about 7 days without a visit. Home Screen web apps are exempt, so always use the icon, not a Safari tab.
- Deleting the Home Screen icon deletes its data. So does *Settings → Safari → Clear History and Website Data*, which can also clear it.
- Back up anyway. After 7 days without a backup, the Backup card in Settings turns orange as a reminder. A backup takes two taps.

### Updating the app later

Edit the files, change the version in `docs/sw.js` (`const CACHE = 'fit-tracker-v2'`), and upload or push again. The phone picks up the new version the next time you open the app while online. Sometimes it takes two launches. **Settings → Check for app update** forces a check. Your data isn't touched by updates.

---

## Backup, restore and using two devices

- **Settings → Save backup file** downloads a `.json` file. **Share / AirDrop** opens the iOS share sheet, so you can save to Files or iCloud Drive, AirDrop it, or email it.
- **Settings → Import a backup** checks the file first. Corrupt or unrelated files are rejected without changing anything. Older backup formats are upgraded automatically.
  - **Merge (recommended)** combines both copies. For any day edited on both devices, the most recent edit wins. Workouts from both are kept, and deletions carry over.
  - **Replace** makes this device an exact copy of the backup.
  - A copy of your data from just before the import is kept in browser storage either way.
- **Sync** is manual: export on one device, import with Merge on the other. Automatic sync would need a server, which this app deliberately doesn't have.

---

## Editing the plan

All plan data lives at the top of `docs/index.html`, in the first `<script id="core">` block, separate from the logic:

| What | Where |
|---|---|
| Exercises | `E('id', 'Name', 'reps'/'time', area, 'focus,list', tier, MET, [start, end dose], [poseA, poseB], 'cue', extras)` lines |
| Original days 1, 2, 3, 30 | `SEED_DAYS` |
| Session templates (A cardio & core, B mat core, C cardio & planks, F finale) | `TEMPLATES`. Each has warm-up, core pool and cool-down lists per stage (early, mid, late, advanced) |
| Dumbbell sessions | `DB_TEMPLATES`, `C1_FINISHERS` |
| Cycle levels, ramp, rest scaling, VR / game minutes | `CYCLES` |
| Games you can log on VR / game days | `VR_GAMES` (plus a MET default in `defaultSettings()`) |
| Stick-figure poses | `POSE_DEF` |

After editing, run `node tests/run-tests.js` (Node 18+) to confirm days 1/2/3/30 are untouched and the rules still hold.

You can also edit a single day inside the app: open the day, tap the **pencil**, and reorder, swap, remove, add or change targets. **Reset to plan** undoes it.

**Your own media:** put files at `docs/media/<exercise-id>.mp4` or `.jpg` (ids are the first argument of each `E(...)` line), then turn on **Settings → Use my own media files**.

---

## How the plan works

- Each cycle is **30 days plus a rest day** (day 31), so there are never more than 3 training days in a row, even across cycles.
- Each 4-day block is **workout → VR / game → workout (cycle 1) or strength (cycle 2+) → rest**. The VR / game day sits between harder days, so bodyweight core sessions are never back to back.
- **Cycle 1:**
  - Days 1–3 are the seeded sessions with no VR / game day, so you learn the moves first. The first one is day 6.
  - From day 9, workouts end with a 2-move dumbbell finisher.
  - From about the halfway point, the core section becomes a 2-round circuit.
  - Day 30 is the seeded finale.
- **Cycles 2–4** (Intermediate, Advanced, Peak):
  - Each block has a core session, a VR / game session and a dumbbell strength session (two alternating types).
  - Core sessions get harder movements (side planks → T planks, sit-ups → V-ups, push-ups), more circuit rounds, higher volume and shorter rests.
  - Strength reps run 8 → 12 through each cycle.
  - Cycle 5 onward repeats Peak.
- **Missed days:** Today offers **Pick up Day N** (shifts the rest of the plan back) or **Skip and continue**. There's also **Move it to tomorrow** on today's card.

### After the first plan: phases

The app is built to keep going after the first goal. A **phase** is one goal plus one run of the plan.

- **Starting one:** go to **Settings → Plan → Start a new phase…**. Today also shows a "Ready for what's next?" card once you reach your goal, pass the goal date, or finish all four levels. **Not now** hides it for two weeks.
- **What you choose:**
  - **Goal:** *Lose more* (a new goal weight and date) or *Maintain* (a target weight with no deadline).
  - **Start date:** defaults to today.
  - **Level:** Beginner, Intermediate, Advanced or Peak. It defaults to one above where you are, so the workouts keep progressing rather than repeating.
  - **Focus:**
    - *Balanced* keeps today's shape: core-first workouts, a VR / game day and a strength day in each block.
    - *Strength* keeps the same block shape (core day, VR / game day, strength day, rest), but each strength day gets an extra set and an extra exercise, using heavier 6–10 rep work. The app suggests a heavier dumbbell once you hit 10 reps. That's good for holding on to muscle once you're near goal weight.
- **What's kept:** all weigh-ins, workouts, streaks and calendar history. The old phase is listed in Settings with its dates and start → end weight. The chart keeps its goal line faintly and marks where the new phase began.
- **What's new:** the Progress card and pace measure from the new phase's start. Each phase has its own done-ticks and day edits, so redoing a level doesn't show as already done.
- **Maintain mode:** the Progress card shows your trend against the target. It reads "Holding steady" within ±1 kg, and "Above range" or "Below range" outside that. The chart shades the ±1 kg band. Pace and deadline numbers are hidden because they don't apply.

### Adding games and activities

On any VR / game day, tap **+ Add a game or activity** (just above *Done: log it*). Give it a name, an optional type, and pick Light, Moderate or Hard; that sets the energy estimate. It appears straight away with its own timer, on every future game day too. You can fine-tune or remove added activities in **Settings → VR / games and activities**. Removing one hides it from the list, but past logs keep its name.

---

## Checklist to verify on the iPhone

I couldn't test on a real iPhone, so please check these:

- [ ] **Home Screen launch:** the icon shows the orange flame and the app opens full-screen with no Safari bars.
- [ ] **Offline:** in airplane mode, the app still opens from the icon.
- [ ] **Storage:** log a weight, swipe the app away, reopen it, and the weight is still there. Check again a few days later.
- [ ] **Speech:** tap Start on a workout and you hear "Get ready. First up: …". If you hear nothing, check the silent switch and volume. **Settings → Voice → Test voice** tests it separately. Pick an "(on-device)" voice if you see several.
- [ ] **Speech with the screen locked:** I expect it to **stop**. iOS pauses web pages when the screen locks, so the app pauses the workout when it goes to the background and shows "Paused" when you come back. Please confirm.
- [ ] **Wake lock:** during a workout the screen should stay on. Older iOS versions may not support this. If the screen dims, set *Settings → Display → Auto-Lock* to a longer time while training.
- [ ] **Vibration:** iPhones don't support web vibration, so beeps and voice are the cues there. That's expected.
- [ ] **Backup:** Share / AirDrop shows the share sheet, and the file opens on the PC and imports.
- [ ] **Game stopwatch:** start a timer, lock the phone for a few minutes, unlock it, and the time has kept counting.

---

## Decisions to double-check

1. **VR / game placement:** one VR / game day in the middle of each 4-day block. None in cycle 1's first block (learning the seeded days). That gives 6 of them in cycle 1 and 7 per later cycle. Targets are 25 / 30 / 35 / 40 minutes by cycle.
2. **Rest day between cycles:** each cycle is 31 days. Four cycles take about 124 days, so a four-cycle run can end a few days after your goal date.
3. **Cycles 2–4 progression:** step counts go from 19–25 to 23–29, doses run from 85% to 125% of the cycle-1 day-30 values, circuits go from 2 to 3 rounds, rests scale to 80/70/60%, and strength sets go 2→3 then 3→4. Treat these as a sensible starting point and adjust `CYCLES` if they feel too easy or too hard.
4. **Added exercises (44):** squats, sumo squats, reverse lunges, wall sit, calf raises, glute bridge march, single-leg bridges, knee/incline/full/pike push-ups, shoulder taps, superman, reverse snow angels, prone Y-T raises, hollow hold, flutter kicks, reverse crunch, plank jacks, squat thrusts, V-ups, march in place, butt kicks, skater hops/steps, shadow boxing, arm circles, child's pose, cat-cow, hip flexor / hamstring / shoulder stretches, and 9 dumbbell moves. Core stays the majority of cycle 1. Later cycles mix in more legs, upper body and back.
5. **Low-impact swaps:** jumping jacks → side step jacks, skipping / high stepping / butt kicks → march, mountain climbers → slow mountain climbers, skater hops → skater steps, plank jacks → shoulder taps, squat thrusts → squats, high knee twist → standing bicycle crunch. Off by default, as you chose.
6. **Energy estimates (shown in kJ):**
   - Seeded days use the original app's numbers, scaled by your weight against a 99 kg reference.
   - Generated days use per-exercise MET values × weight × time, plus rests. A 1.15 calibration factor brings them in line with the original numbers (about 350 kJ for the first generated day, about 620 kJ for day 29).
   - Energy is calculated in kcal and shown in kJ (1 kcal = 4.184 kJ, rounded to the nearest 10). **Settings → Display → Energy units** switches to kcal if you ever want it.
   - VR / game days use MET 5.5 for Beat Saber, 7 for Body Combat, 5 for Just Dance and 4.5 for Ring Fit Adventure, all editable in Settings. To add another game, add a line to `VR_GAMES` and a MET default in `defaultSettings()`.
   - All are labelled estimates.
7. **Pace status:** actual pace is a least-squares fit over the last 3 weeks of weigh-ins. "Ahead" means at least 110% of the required pace, "On track" 85–110%, otherwise "Behind". There's a caution line whenever more than 1 kg/week is required.
8. **Streak:** counts completed training days. Rest and moved days don't break it, and today doesn't break it until it's over.
9. **Messages on Today:** a short morning, afternoon or evening line based on what's planned and what you've logged. It's picked from a small fixed set, so it doesn't repeat randomly within a day.
10. **Next-phase defaults:** the level defaults to one above your current one, capped at Peak. If you're within 0.5 kg of your goal, the goal type defaults to Maintain, with your current trend weight as the target. Maintenance counts ±1 kg of trend weight as "holding". A strength-focus phase keeps a core day and a VR / game day in every block; only the strength days change (one extra set, one extra exercise, 6–10 reps).
11. **Added activities' intensity:** Light, Moderate and Hard map to MET 3.5, 5 and 7.

---

General fitness guidance, not medical advice. Stop if something hurts.
