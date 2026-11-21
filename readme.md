# IntelBot & CI Bot

Two Discord bots, written in Python with [discord.py](https://discordpy.readthedocs.io/) in 2023–2024, for counter-intelligence work in EVE Online alliance Discords: finding out who is leaking channels to other groups through relays, and making day-to-day member management easier.

| Script | What it is |
| --- | --- |
| [`cibot.py`](cibot.py) | **CI Bot**: slash commands for relay hunting (`/burnrelay`), role and permission tools, member search, and EVE character lookups. |
| [`intelbot.py`](intelbot.py) | **IntelBot**: a passive watcher. It forwards DMs sent to the bot into a channel and reports members who are active on the Discord **web client**. |

> [!WARNING]
> **The CDN relay-burning technique described below no longer works.** It relied on how Discord fetched link embeds in 2023. Discord has since changed how its CDN and Cloudflare handle those fetches, so a relayed link no longer causes a second request to your server. The write-up is kept here for historical interest. The rest of the bots still run, but they are archived and not actively maintained.

---

## Background: how relay burning worked

A *relay* is a self-bot or bot account that sits in your Discord and copies messages from your channels into a hostile server.

### The CDN embed bug (now patched)

In 2023, Discord fetched the image behind an embedded link separately for each guild it was shown in, instead of fetching it once and caching it.

So if you hosted an image on your own web server (say `smug.jpeg`) and posted the link in a channel that was being relayed, your server would see **two** requests: one for your guild and one for the guild the relay posted into. Seeing a second fetch confirmed that the channel was being relayed.

Most relays tried to block this by putting a zero-width character in links so they would not embed. They did **not** clean image URLs inside embeds, so wrapping the image in an embed got past that protection.

Discord's CDN and Cloudflare setup has since changed how these fetches are handled, so the duplicate request no longer happens and this detection method is dead.

### Narrowing down the leaker (`/burnrelay`)

Once you know a channel is relayed, you still have to find *who* is relaying it. Most relays turn themselves off when the account's roles or channel permissions change, but you can sometimes change channel settings in a way that forces them to stay on.

`/burnrelay` then goes through every member with the `Member` role, one at a time:

1. Gives that one member permission to view the current channel.
2. Posts their display name, Discord ID, and a random 4-digit number (1000–9999).
3. Writes the same line to a private log channel.
4. Removes the permission and deletes the message.

If one of those lines turns up on the hostile relay, that member's account is the one relaying. The random number lets you check the line against your log channel, so nobody can fake a result by making up a name.

---

## CI Bot (`cibot.py`)

### Slash commands

| Command | What it does |
| --- | --- |
| `/burnrelay` | Goes through every member with the `Member` role, as described above. Every line is also written to the log channel. |
| `/copy_permissions <source> <target>` | Copies every role and member permission overwrite from one text channel to another. Useful for setting up a temporary decoy channel to catch a relay. |
| `/split_role <name_1> <name_2> <role>` | Creates two new roles and splits the members of an existing role evenly between them. Run it repeatedly to narrow a group down without assigning roles by hand. |
| `/search <ticker>` | Lists every member whose nickname contains `<ticker>` (for example, a corp ticker). Members with `Corporation Leadership` or `Alliance Execs` are shown in **bold**, with a `(cap)`, `(scap)`, or `(cap/scap)` tag based on their `CAP-*` roles. |
| `/lookup <character_id>` | Looks up an EVE character on ESI and zKillboard: name, birthday, corp, alliance, and kills/losses. Characters with more than 30 kills plus losses are marked as recruitable. Names in the hard-coded blacklist are flagged and never recommended. |

### Prefix command

| Command | What it does |
| --- | --- |
| `$sync` | Re-syncs slash commands with Discord. Only members with the `Admin` role can use it. `$sync` syncs globally, `$sync ~` syncs to the current guild, `$sync *` copies global commands to the current guild, and `$sync ^` clears the guild's commands. |

### Background events

- **Leaver alerts.** When a member who has one of the roles in `group_names` (by default `CAPS - Capitals` and `CAPS - Supers`) leaves the server, you are sent a DM.
- **DM forwarding.** Any DM sent to CI Bot is forwarded to you, including attachment links.

---

## IntelBot (`intelbot.py`)

IntelBot has no commands. It just listens.

- **Web client watch.** When it starts, IntelBot posts a report of every member who is currently online, away, or on Do Not Disturb on the Discord **web** client. After that, it posts an embed whenever someone's web or desktop status changes while they are active on web. If someone is **online on web but idle on desktop**, which is a common sign of a second session left open to watch channels, it pings `@everyone`.
- **DM relay.** Any DM sent to IntelBot is reposted as an embed in the relay channel, with the sender's name, avatar, and a timestamp.

---

## Setup

### 1. Create the bot accounts

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications) and create an application for each bot you want to run.
2. Under **Bot**, copy the token.
3. Under **Bot → Privileged Gateway Intents**, turn on **Presence Intent**, **Server Members Intent**, and **Message Content Intent**. Both scripts use `discord.Intents.all()` and will not connect without them.
4. Under **OAuth2 → URL Generator**, select the `bot` and `applications.commands` scopes and the permissions the bot needs (CI Bot needs Manage Roles, Manage Channels, and Manage Messages). Then use the generated URL to invite the bot to your server.

### 2. Install

You need Python 3.8 or newer.

```bash
git clone https://github.com/ru-dash/IntelBot.git
cd IntelBot
python -m venv venv

# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

pip install -r requirements.txt
```

### 3. Configure

Settings are hard-coded near the top or bottom of each script. Edit them before you run the bot.

**`intelbot.py`**

| Setting | Set it to |
| --- | --- |
| `BOT_TOKEN` | Your bot token |
| `RELAY_CHANNEL_ID` | The channel where DMs to the bot are reposted |
| `PLATFORM_UPDATE_CHANNEL_ID` | The channel for web client reports (it can be the same channel) |

**`cibot.py`**

| Setting | Where | Set it to |
| --- | --- | --- |
| `bot.run('TOKEN')` | last line | Your bot token |
| `log_channel_id` | inside `burnrelay` | The private channel that logs `/burnrelay` output |
| `target_user_id` | above `on_member_remove` | Your Discord user ID. Leaver alerts and forwarded DMs go here. |
| `group_names` | above `on_member_remove` | The roles that trigger leaver alerts, and how each is described in the alert |
| `blacklist_array` | inside `lookup` | The character names to flag as blacklisted |
| Role names | throughout | `Member`, `Admin`, `Corporation Leadership`, `Alliance Execs`, `CAP-*`. Rename these to match your server's roles. |

To get channel and user IDs, turn on **Developer Mode** in Discord (User Settings → Advanced), then right-click a channel or user and choose **Copy ID**.

### 4. Run

```bash
python cibot.py      # CI Bot
python intelbot.py   # IntelBot
```

Each bot runs on its own, so you can run one or both, each in its own terminal. CI Bot syncs its slash commands when it starts. Global commands can take a while to show up in Discord, so if you want them right away, run `$sync ~` in your server.

---

## Notes

- Earlier versions of this README listed a few features that are **not** in the current code: alerts for accounts younger than 90 days, and Mumble certificate hash flagging (that lives in [Mumble-Relay](https://github.com/ru-dash/Mumble-Relay)).
- Bot tokens are kept in the source files. Never commit a real token. If one leaks, reset it in the Developer Portal right away.
