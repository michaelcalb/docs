---
title: validation
---
[[index|🏠 Return to Hub]] [🐈‍⬛ Project Repository](https://github.com/michaelcalb/osu-rmc)

---

This file aims to explain how the validation system works, both for you (the player) and for me (the guy who made this), in case I forget how it works in the future, which is very likely to happen.

# statuses
A run can have 6 different statuses (one at a time):
- **active** means the run is still being played
- **pending_final** means the run has ended but validation hasn't started yet (it may be waiting for you to start it, or waiting to be retried)
- **validating** means the run is going through the validation process
- **complete** means the run validated successfully and is safe
- **void** means the server kept failing to validate the run for 6 hours and gave up. Voided runs can still be recovered
- **abandoned** means the run was abandoned (or deleted) by the player and will be automatically deleted once it's more than an hour old

# triggering the validation
A run can end in three ways, and all of them call the same function in the code: `finishRun()`, which changes the run's status to `pending_final`. What happens next depends on how it ended.

- Pressing the "end run" button on the active run page ends the run and starts the validation automatically
- Theoretically, the run can also end by running out of maps to play. Realistically, that's impossible unless you spam the next map button without actually playing. It works exactly like pressing "end run"
- When the run's main timer (1h) hits zero, the run ends, but the validation does **not** start on its own. You'll see a "time is over!" screen with two buttons: "start validation" and "delete run"

The reason for that last one is simple: when the timer runs out, you might still be playing the last map, and that map still counts (more on that in [[#step 8. the last map|step 8]]). The server has no way of knowing when you're done, so it waits for you to tell it. Finish the map, then press "start validation".

If you don't press either button, the run is deleted after 8 hours. While a run is waiting for your decision, you can't start a new one.

The validation process itself is exactly the same in every case.

# validating the run

## step 0. wait a second
During the whole run, no verification is done on the player's input. The server doesn't know if their FC claims are true or if they marked perfectly playable maps as "unplayable" to skip them. The server doesn't run those checks in real time because each one costs an osu! API call, and osu!'s API is limited to roughly 1 request per second (osu!RMC plays it safe and sends at most 1 request every 1.5 seconds).

With that in mind, I decided to take an approach similar to how Trackmania validates its players' runs. To compete on the leaderboards, players have to upload their replay files (basically a bunch of text with their inputs) to the website, and the validation is then done based on that single file, **once**.

osu!RMC doesn't validate your inputs in real time, but it still stores all of them in its database. After the run ends, it compares all of your "claimed" outcomes with what osu! itself has stored in its database, and for that, it only needs **one** osu! API call. One is enough under perfect conditions, but the number can grow because of:
- Server instability (on both ends)
- Maps claimed as "unplayable" during the run (+1 request per map)

When a run ends, its status becomes `pending_final`.

## step 1.1. wait 10 seconds
Before the validation truly starts, the server makes sure at least 10 seconds have passed since the run ended. This exists to make sure all scores set by the player are already on osu!'s servers and can be seen publicly. This timer is enough >99% of the time, while the other <1% is covered by other measures.

If you pressed "start validation" on the "time is over!" screen, those 10 seconds have almost certainly passed already, so the validation starts immediately.

## step 1.2. wait 1 minute
Independently of any timer, a function runs on the server **every single minute**, forever. It does two things in order:

First, it checks for any run that has had the `validating` status for more than 5 minutes. At this point, it's very likely some error occurred during validation. Those runs return to the `pending_final` status so the validation process can restart.

Second, it finds every run with the `pending_final` status whose validation was requested more than a minute ago (but less than 10 minutes ago), and tries to validate each one. This "more than a minute ago" filter exists so it doesn't conflict with the first 10 seconds after the run ended. It also serves as a fallback for when the 10-second timer fails.

Runs that ran out of time and are still waiting for the player to press "start validation" are ignored here, since nothing was requested yet.

## step 1.3. wait 1 hour
If a run is still `pending_final` 10 minutes after its validation was requested, something is going wrong (most likely osu!'s servers being unstable). Instead of hammering the osu! API every minute, the server backs off and retries those runs **once every hour**.

If a run still fails to validate 6 hours after its validation was requested, the server gives up and the run enters [[#the void|the void]].

## step 2. all-in
This step can be reached through steps 1.1, 1.2 or 1.3, whichever happens first.

Here, the status of the run is changed from `pending_final` to `validating`, along with a new `validating_since` value, which stores when the validation began.

Now, the validation has truly begun.

If anything fails from here on, the run immediately goes back to `pending_final`, and steps 1.2 and 1.3 will pick it up again later.

## step 3. loading the maps
Every run has a total of 60 maps, but not all of them are always served to the player[^1]. So we have to gather the ones the player interacted with (FCed, skipped, etc.). This step loads the following information for each served beatmap:
- `map_id`: which beatmap it was
- `served_at`: the exact moment the server handed the map to the player
- `claimed_outcome`: what the player said happened to the map

## step 4. talking to osu!
Here, the server calls the osu! API for the player's recent scores list. This is the **only** source of truth. Up to this point, everything was reported by the client (player), and the client should never be trusted.

The "recent scores" endpoint returns all[^2] osu!standard scores submitted by the player in the last 24 hours. Failed plays are not included.

## step 5. mapping the maps
Now the server has 2 lists: the maps served during the run and the scores fetched from osu!, which is our source of truth. This step consists of comparing both lists to verify the claims made by the player.

For every map, the server looks in the scores list for plays that satisfy two conditions:
- the beatmap ID matches
- the play finished after the map was served

If **both** conditions are **true**, it means the server found a score set during the run. If multiple plays satisfy both conditions, it looks for an FC. If multiple FCs are found, it takes the one with the highest score. If none is an FC, it takes the play with the best rank (SS > S > A > B > C > D), and if there's still a tie, the one with the highest score.

## step 6. mapping the times
Although the server now has confirmation that each score was set after its map was served to the player, it still needs to verify whether all those scores started within the run's 1-hour timer.

Also, the client (player) could have abused the preparation timer to gain a few extra seconds on the run. The purpose of this extra 60-second timer between maps is to give the player enough time to go to the beatmap's page, download it, open it in osu! and start playing. This way, players with slow internet connections or low-end computers aren't put at a disadvantage by losing time on the 1-hour timer. However, most players, even with those issues, won't use the whole minute, and starting to play the map doesn't swap the timers automatically.

That's why this calculation is needed. The server rebuilds the whole run's clock on its own, using only the moment each map was served and the moment osu! says each score started. For every map, up to 60 seconds between being served and starting the score count as preparation time and are free. Anything beyond that comes out of the main, 1-hour timer, and so does all the time after the score started until the next map was served. That means that, during the run, if you didn't click the "start map" button and started maps while the preparation timer was running instead, your run could've lasted a few minutes longer than an hour. This calculation finds which maps were started after the **true 1-hour mark**.

The buttons you press during the run don't matter here, only the timestamps do. What the buttons do is keep the timer you see during the run close to the one the server calculates.

In short, it's highly encouraged to use the buttons in the interface properly ;)

## step 7. unplayable claims
>[!NOTE]
>This is an optional step.

For each map the player marked as "unplayable", the server calls the osu! API for that beatmap's current status.

This step exists because the map pool used in osu!RMC is not synchronized in real time with osu!, meaning maps the server sees as ranked and completely playable could have been deleted or unranked since.

The claim holds if the map was deleted (or can't be found anymore), had its download disabled, or is no longer ranked, approved, qualified or loved. The maps are checked in order, and the server stops at the first false claim, since the run will be cut there anyway.

## step 8. the last map
By the rules of osu!RMC, if you started a map within the time limit, but your FC only landed after the time was over, that map still counts for your run. That's why a run that ran out of time waits for you to press "start validation" (see [[#triggering the validation|triggering the validation]]).

Apart from free-skipped and unplayable maps, the last map served is also the only map allowed to have no score at all. You were probably still downloading it, or mid-play, when the run ended. It is recorded as an incomplete final map, it doesn't count as an FC, and it doesn't cut the run.

## step 9. the verdict
All the maps are verified in order by going through a sequence of questions. For each map, the first question that applies decides what happens:

1. **Does the score use a banned mod?**
	The banned mods are Autoplay, Cinema, Relax, Autopilot and Spun Out. This applies even to maps you free-skipped or claimed unplayable, if you played them.
	If so, **the run is cut**.
2. **Was it free-skipped?**
	Record it as a `free_skip` and move on. It can only be used **once** per run, and the server already enforces that during the run itself: a second free skip is simply refused.
3. **Was it claimed unplayable?**
	If [[#step 7. unplayable claims|step 7]] showed the map is **playable**, the claim was false and **the run is cut**.
	Otherwise, record it as `unplayable` and move on.
4. **Is there no score at all?**
	If this is **not** the last map, **the run is cut**. If it is, see [[#step 8. the last map|step 8]].
5. **Does the score use any other mods?**
	If the score has any allowed[^3] mod, the run goes to the `free_mod` leaderboard, no matter which mode you picked when starting it. This applies even if the run gets cut on this same map by questions 6 to 8. If you reached this question, your mods count. If **no** score in the run uses one, it goes to the `no_mod` leaderboard. Some mods still count as no mod: No Fail, Accuracy Challenge, Perfect, Sudden Death, Strict Tracking, Classic, Mirror, Alternate, Single Tap and Touch Device.
6. **Is it a full combo?**
	Zero misses and zero slider breaks. If [[#step 6. mapping the times|step 6]] said you started it after the true 1-hour mark, **the run is cut**.
	Otherwise, it counts as an FC.
7. **Is it an A rank or better, but NOT an FC?**
	Record it as an `a_skip` and move on.
8. **Anything else**
	If the score reached this far, **the run is cut**.

"Cutting" the run means the verification loop stops immediately. Everything verified before that point is kept and counts as valid.

## step 10. the hall of fame
Lastly, your run is written into our database, along with its FC count, average FC score and average FC accuracy. Its status becomes `complete`.

Congratulations! If your run reached this far, it's safe in osu!RMC's database and will be displayed on your profile. If it has at least one FC, it can also show up on the leaderboards, which display each player's best run. Time to brag about it ;)

## the void
The spooky part of validation.

A lot of unintended errors can happen during the validation. Some of them can be prevented or worked around, some can't. When an error happens, the server simply tries again later (see [[#step 1.2. wait 1 minute|step 1.2]] and [[#step 1.3. wait 1 hour|step 1.3]]). If it keeps failing for 6 hours, your run won't be validated, but it also **won't be lost**. Instead, it will enter the `void` state. Voided runs aren't displayed on the leaderboards or your profile. They only exist in the database, waiting to be fixed.

If your run was voided and you care about it, please contact me, preferably through Discord, so we can fix it ;)

[^1]: In fact, I believe they never will be in a real, serious run.
[^2]: Technically not "all" scores, only the latest 100. That's way more than anyone can set in an hour.
[^3]: Any mod that isn't on the banned list in question 1.
