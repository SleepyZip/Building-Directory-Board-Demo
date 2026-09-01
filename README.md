# Directory Board Demo

A self-contained, data-driven lobby directory board built for unattended TV kiosks — portrait auto-scaling, self-refreshing, zero backend.

**[Live demo →]([#](https://sleepyzip.github.io/Building-Directory-Board-Demo/))**

## What is this?

A single HTML file renders a full multi-floor building directory from one plain JS data object. no templates, no build step, no server. It's designed to run on a PC connected to a display in a lobby, 24/7, unattended, refreshing itself to pick up changes without anyone touching the display.

The building, tenants, and staff shown ("Beacon Ridge Commons") are entirely fictional. This is a sanitized demo built from the same rendering engine used in a real board originally developed for a multi-tenant office building, with all identifying details replaced for portfolio purposes.

## Features

- **Data-driven layout** — the entire board (building info, floors, suites, staff rosters) is one JS object; editing it is the only thing needed to update what's on screen.
- **Portrait auto-scaling** — Scales to fit whatever screen or aspect ratio it lands on.
- **Self-refreshing** — reloads itself on a timer to pick up data edits with no server, no push, no manual restart.
- **Burn-in guard** — a subtle pixel-shift nudge on a timer, since this is meant to sit on the same TV for months at a time without a restart.
- **Smart credential parsing** — `"Last, First, CRED"` entries are auto-split and alphabetized, so lists don't need manual sorting.
- **Real client-side QR generation** — no external image dependency, generated on load.
- **Configurable header treatment** — logo or typeset wordmark, light or dark header, swappable via config.
- **Zero backend** — plain HTML/CSS/JS; deploy it anywhere that can serve a static file.

## Tech

Vanilla HTML/CSS/JS. One external script (a QR-code generation library, loaded from a CDN) — everything else is dependency-free, no framework, no build step.

## Customizing

Everything lives in the `BOARD` object near the top of the `<script>` block:

- `building` — name, address, header text
- `logo` — swap in an image or fall back to the typeset wordmark; toggle light/dark header
- `floors` — suites, directional arrows, staff rosters, per-suite notes
- `wifi` — guest network copy (the QR code encodes whatever text is set here)
- `canvas` — set this to match the target screen's actual resolution
- CSS custom properties in `:root` control the entire color palette

## Running it

Open the HTML file directly in a browser — no build step, no install. For an actual kiosk deployment, run it in a browser's kiosk/fullscreen mode on a PC connected to the display.

## Scheduled Task Setup (Production Deployment)

Viewing the file in a browser is enough to see the demo, but the board is designed to run **unattended, 24/7, on a dedicated kiosk PC** — which means it needs to survive reboots, come back after a power cycle, and periodically reset itself rather than accumulate memory/rendering issues from days of uninterrupted uptime. On Windows, that's handled by pairing the page with a Scheduled Task rather than just leaving a browser tab open.

The task does three things:

1. **Launches Chrome in kiosk mode** pointed at the local HTML file (not a URL — the page has zero server dependency, so it runs even fully offline)
2. **Fully restarts the browser process on an interval** (e.g. every 10 hours) — a heavier reset than the page's own self-refresh, which only reloads the DOM. This is what actually clears out any long-uptime browser-level issues.
3. **Re-launches on every logon**, so a reboot or power cycle recovers automatically with no one needing to touch the machine

A minimal version of the launch + scheduling logic:

```powershell
# Launch-Kiosk.ps1 — (re)start Chrome in kiosk mode against the local file
$chrome = "$env:ProgramFiles\Google\Chrome\Application\chrome.exe"
$url    = "file:///C:/Kiosk/directory-board-demo.html"

Get-Process chrome -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Sleep -Seconds 2

Start-Process -FilePath $chrome -ArgumentList @(
    "--kiosk", "`"$url`"",
    "--no-first-run",
    "--noerrdialogs",
    "--disable-infobars",
    "--disable-session-crashed-bubble",
    "--user-data-dir=`"C:\Kiosk\ChromeProfile`""
)
```

```powershell
# Register the scheduled task: starts at 5:00 AM, repeats every 3.5 hours
# indefinitely, plus relaunches on every logon.
$action = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -WindowStyle Hidden -File "C:\Kiosk\Launch-Kiosk.ps1"'

$triggerRepeat = New-ScheduledTaskTrigger -Once -At (Get-Date -Hour 5 -Minute 0 -Second 0) `
    -RepetitionInterval (New-TimeSpan -Hours 3 -Minutes 30)
$triggerRepeat.Repetition.Duration = ""   # empty = repeat indefinitely

$triggerLogon = New-ScheduledTaskTrigger -AtLogOn

Register-ScheduledTask -TaskName "Directory Board Kiosk" `
    -Action $action -Trigger @($triggerRepeat, $triggerLogon) -Force
```

A few things learned the hard way building the original deployment, worth keeping in mind if you adapt this:

- **`-RepetitionDuration` has a max.** Passing `[TimeSpan]::MaxValue` to express "repeat forever" fails registration outright (`Duration is incorrectly formatted or out of range`) — an empty string on `Repetition.Duration`, matching the Task Scheduler GUI's "Indefinitely" option, is the correct way to do it.
- **Force-killing Chrome can leave a stale `SingletonLock`** in its profile folder, which then silently blocks the *next* launch from opening a window at all. Clear `SingletonLock` / `SingletonSocket` / `SingletonCookie` from the profile directory before each relaunch.
- **A kiosk window needs an interactive desktop session** — the task can't run invisibly as SYSTEM. The PC needs to be configured to auto-logon (e.g. via Sysinternals Autologon) so a session already exists after any reboot or power cycle.
- **Log every step to a file, not just the console.** Once this runs hidden via Task Scheduler, there's no console to read — a simple timestamped log next to the script turns "it's just not working" into an actual diagnosis.


