# security notes

notes on the security work in reysbot, a self-hosted discord bot and admin panel i run for my own server. the platform handles role-based access to admin functions, a minecraft server behind a proxy, a chat relay between discord and the game, and an in-server economy.

everything below is a real incident or a real control. the code is private because it holds live config, so this is the writeup instead.

## what the system is

- a python discord bot on a proxmox lxc
- a web admin panel with its own auth, scoped permissions and an audit log
- a minecraft server on a separate vm, reachable only through a velocity proxy on a hetzner vps
- a wireguard tunnel between the vps and the minecraft vm
- rcon and ssh to the game server, used by the bot

the bot can time people out, purge channels, move money, whitelist players, read inventories, and restart the game server. so the interesting surface is not the internet, it is the panel and who can press what.

## permissions

### a channel went public because a neutral overwrite is not a deny

i built a setup command that creates the minecraft channels. after running it, `#mc-control` was readable by everyone. that channel can restart the server.

two things caused it. first, discord's `set_permissions(..., view_channel=None)` writes a neutral overwrite, it does not remove the overwrite. neutral means "inherit", and inherit resolved to allow. second, categories do not live-inherit to children. a channel created under a private category keeps whatever it was given at creation time, so locking the category down afterwards changed nothing.

what changed:

- every channel in the setup has an explicit intent recorded in code, staff-only or public. nothing is implicit.
- a never-widen rule. the setup can tighten permissions and can create channels, it cannot broaden an existing channel's visibility. if it wants to, it fails and says so.
- the command now prints a plan and waits for `!confirm`. the plan names every channel it will touch and what the permissions will be after.
- the resulting permissions are written to the audit log, not just the intent.

the general lesson is that "i set it to none" and "i denied it" are different operations and the api does not stop you from confusing them.

### scopes are declared and the declaration is tested

every admin route declares the scope it needs. a parity test walks the route table in both directions: a route with no declared scope fails, and a declared scope with no route fails. adding an endpoint without deciding who can call it breaks the build.

this is not clever, it just has to be enforced by something other than memory. the panel has grown to a few dozen routes across tickets, economy, moderation, streams, voice and minecraft, and i was not going to keep it straight by hand.

the generalized version of this check is published separately as [scope-parity](https://github.com/ReyDotExe/scope-parity).

## identity

### display names are not identifiers

minecraft display names carry scoreboard team prefixes. i use teams to show `[OWNER]`, `[ADMIN]` and so on in chat. the prefix is part of the display name, so anything that reads a display name gets the prefix too.

this broke three things over about two weeks, in three different places, for the same reason:

- `/list` returned prefixed names, so the panel's "who is online" never matched anyone
- the inventory snapshot could not resolve a player and silently fell back to the save file on disk, which is stale
- the join and leave parser used a regex anchored on the name, so staff joins and leaves never fired. that also silently stopped playtime earnings for anyone with a role

the fix in each case was to stop using the display name as a key. `list uuids` returns uuids, the snapshot resolves by uuid, and the join parser finds the phrase first and takes the name relative to it rather than assuming the name starts the field.

the broader point is that i was treating a presentation string as an identity. once a prefix system existed, every consumer of that string was wrong, and they failed quietly rather than erroring.

### linking discord accounts to game accounts

whitelisting goes through a ticket. staff press approve in the panel, which resolves the name against mojang's api to get the canonical capitalisation and the uuid, writes the link, and whitelists them. the panel refuses a name already linked to someone else and names the current holder in the refusal.

people who joined before this existed were unlinked. the panel lists whitelisted players with no link, computed from whitelist.json minus the link store, case-folded on both sides so a capitalisation mismatch does not read as "not linked".

## races and state

### the duplicate-action bug

two admins acted on one ticket at the same time and the same action ran three times. the same link got written into the channel three times.

there was a duplicate check. it sat at step eight of a nine-step approve, long after both callers had passed step one. both saw the same "not yet done" and both proceeded.

the fix was to stop checking and start constraining. the seen state goes into the update's where clause, so the database decides who wins. the second caller's update matches zero rows and the caller is told someone else got there first, by name. the panel does this for ticket claims, ticket lifecycle, whitelist writes and the minecraft link. economy grants use a nonce instead of a version, because the thing being protected is a duplicate submission rather than a stale read.

a check that runs before the write is a suggestion. a condition in the write is a rule.

### a failed read looked exactly like an empty one

the panel showed "nobody online" when it could not reach the server, which is the same thing it shows when nobody is online. the footer said "connected" regardless. a hung read showed a loading state forever.

this is the worst class of bug in an admin tool because it is invisible. you make a decision from a screen that is confidently wrong.

the rule i settled on, now enforced across the panel: a read that cannot be performed must never return the same value as a read that found nothing. in practice that means every read path has three outcomes rather than two, requests have timeouts, and the surface says which of the three it is.

### the ui read a field the api never served

a status chip on the dashboard read `d.ready`. the endpoint has only ever served `discord_connected`. undefined is falsy, so the chip rendered red forever on a healthy bot.

my own screenshots showed the red chip. i read it as test data.

the fix has two parts. every chip now declares which fields it reads, and a response carrying none of them renders as "unrecognised answer" and logs the keys it actually got, rather than rendering as a failure. and there is a contract test that calls the real route handlers, no fixtures, and asserts every declared field exists in the real response. it fails in both directions: a chip reading a field nobody serves, and a handler dropping a field something reads.

i ran that test against the commit before the fix. it fails at exactly the incident.

the reason it needed to hit the real handler is that the original test suite passed the whole time. the fixtures were written from the same wrong assumption as the code.

## input handling

### the chat relay

discord messages get relayed into the game with `tellraw`, which takes a json payload that the server parses. a message is attacker-controlled text going into a command interpreter running as the server.

what happens to a message before it leaves discord:

- automod runs first
- section signs are stripped in pairs, so formatting codes cannot be injected
- quotes and backslashes are escaped for the json payload
- mentions are neutralised so a relayed message cannot ping anyone
- length is capped
- rate limited per person and overall

the ordering matters. automod first means a message blocked by discord's own rules never reaches the encoder at all.

## network

### online-mode is off and that is fine, conditionally

the game server runs with `online-mode=false`, because authentication happens at the velocity proxy and the backend uses modern forwarding. on its own that means anyone who can reach port 25565 can log in as anyone.

the control is that ufw on the game vm only accepts 25565 from `10.10.0.1`, the proxy's address on the wireguard tunnel. the proxy is the only thing that can talk to the server, and the proxy authenticates.

this is worth writing down because the safety of the setting lives entirely in a firewall rule on a different host. if that rule is ever lost, nothing about the minecraft config changes and the server becomes impersonatable. it is recorded in the runbook with that reasoning attached, not just as a rule to keep.

the same review found that rcon was not reachable from the bot at all, which explained a long-standing "player list not available" symptom i had been treating as a bug in my own code.

## secrets

the repo auto-pushes. so credentials are documented by name and location only, never by value: the forwarding secret, the rcon password, api keys, totp seeds. the docs say where each one lives and what it is for, and nothing else.

this is currently a discipline rather than a control. a pre-commit hook that blocks anything shaped like a token is the obvious next step and is on the list.

admin accounts on the panel use totp. onboarding a new admin means a temporary password and an enrollment seed sent by dm, with the dm deleted once they confirm they can sign in. the seed is not stored anywhere after enrollment.

## what i would tell someone starting this

most of these were not sophisticated. they were defaults that did not mean what i assumed, checks that ran in the wrong place, and states that rendered identically when they were not identical.

the three that generalise:

1. a permission api that has an "inherit" state will eventually resolve it to allow. say allow or deny explicitly.
2. if two people can do a thing at once, the database has to decide, not your code.
3. a failure and an empty result must never look the same to the person reading the screen.
