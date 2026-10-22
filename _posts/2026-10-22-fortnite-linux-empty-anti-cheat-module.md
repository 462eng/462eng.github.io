---
title: "Fortnite on Linux: the launcher works, the anti-cheat module is an empty file"
date: 2026-10-22 16:10:00 +0200
day: "2026-10-22"
short_title: "Fortnite on Linux: an empty anti-cheat module"
description: "A short note: Epic's new Linux launcher installs Fortnite and starts it with Proton, then Easy Anti-Cheat downloads its Linux module and gets an HTTP 200 with an empty body. That is how Fortnite stays closed on Linux, and the launcher reports it as a crash."
tags: [linux, proton, gaming, easy-anti-cheat, note]
---

**Short version:** Epic's Linux launcher does install and start
Fortnite through Proton. The game then stops because Epic serves an
**empty** Easy Anti-Cheat module for Linux. Nothing can be fixed on
the client side.

What happens, from the launcher's and the Proton prefix's logs:

1. The launcher (an AppImage) installs the Windows build of Fortnite
   and starts it with the GE-Proton it ships. So far, so good.
2. The game launcher hands over to Easy Anti-Cheat. EAC sees it runs on
   Linux (`System name: 'linux64'`) and downloads its Linux module from
   Epic's CDN.
3. `anticheatlauncher.log` then says:

   ```
   Connect result: No error (0) Response Code: 200
   Anti-cheat service disabled on backend.
   Launcher finished with: 204
   Could not reach the Easy Anti-Cheat CDN!
   ```

The last line is misleading: the CDN did answer. Requesting the module
URL by hand returns **HTTP 200 with `content-length: 0`**, a file that
has been in place since May 2023. EAC exits with code 204, the game
launcher with 1, and the Epic launcher shows the session as a crash.

No Proton version, prefix setting or environment variable changes
that. The remaining options are cloud gaming or a real Windows install.
We removed the game and the launcher's data.

The same evening, Rocket League (through Heroic and legendary) also
"did nothing" when launched: no window, no error. The cause was in
legendary's output: `Game is out of date, please update or launch with
update check skipping!`. Updating the game from Heroic fixed it.
Skipping the check would start a version the online servers refuse
anyway.

{% include coauthors.html %}
