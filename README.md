# tgsnap

`tgsnap` is a small Telegram snapshot CLI built with [Telethon](https://github.com/LonamiWebs/Telethon).

It takes point-in-time snapshots of Telegram users, groups, and channels, displays the collected information in the terminal, and stores each run locally without overwriting previous snapshots.

`tgsnap` only collects information Telegram exposes to the authenticated account. It does not join groups or channels, request membership, read messages, mark stories as seen, or increment post views.

## Features

- Automatically detect and resolve:
  - usernames and `t.me` links
  - phone numbers in international `+...` / `00...` form
  - Telegram numeric IDs
  - private group/channel invite links
- Snapshot Telegram users:
  - name and username
  - user ID
  - bio
  - birthday, when available
  - verified, premium, bot, scam, and fake flags
  - personal channel ID, when available
  - profile-photo history
  - public/fallback profile photo
  - profile-music metadata
- Snapshot groups and channels:
  - title and username
  - Telegram ID
  - about text
  - member/admin counts when exposed
  - linked chat ID
  - forum and slow-mode information when applicable
  - visible photo history
- Inspect private invite links without joining or requesting access
- Keep every run as a separate timestamped snapshot
- Track known usernames across snapshots
- Display saved snapshots offline with `--show`
- Show live lookup/download progress without clearing terminal history
- Respect the `NO_COLOR` environment variable

## Requirements

- Python 3.14 when using the included `Pipfile`
- A Telegram account
- Telegram `api_id` and `api_hash`

Create API credentials from [my.telegram.org/apps](https://my.telegram.org/apps).

> `tgsnap` uses Telegram's MTProto user API through Telethon. It does not use a BotFather bot token for normal lookups.

## Installation

Clone the repository and install the dependencies:

```bash
pipenv install
pipenv shell
```

Make the CLI executable if needed:

```bash
chmod +x tgsnap
```

You can also install from `requirements.txt` in a regular virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Configuration

Copy the example environment file:

```bash
cp .env.example .env
```

Then set your Telegram API credentials:

```env
TELEGRAM_API_ID=12345678
TELEGRAM_API_HASH=0123456789abcdef0123456789abcdef

TELEGRAM_SESSION=tgsnap
TGSNAP_OUTPUT_DIR=~/.local/share/tgsnap
```

`TELEGRAM_SESSION` is optional. Telethon stores the authenticated login state in a local `.session` file so later runs normally do not require another login.

`TGSNAP_OUTPUT_DIR` is also optional. If it is unset, snapshots are stored under `./output`.

The command-line `--output` option overrides `TGSNAP_OUTPUT_DIR`.

## Usage

### User by username

```bash
./tgsnap username
./tgsnap @username
./tgsnap https://t.me/username
```

### User by phone number

Phone numbers are detected automatically when they begin with `+` or `00`:

```bash
./tgsnap +15551234567
./tgsnap 0015551234567
```

If the number is not already in your Telegram contacts, `tgsnap` temporarily imports it for resolution and removes the temporary contact afterward.

### Telegram numeric ID

```bash
./tgsnap 123456789
```

Numeric IDs can only be resolved when the authenticated account/Telethon session has enough information to identify that entity. `tgsnap` loads the account's dialogs once and retries if necessary.

### Public group or channel

```bash
./tgsnap https://t.me/channelname
./tgsnap @groupname
```

The resolved Telegram entity determines whether the target is stored as a user, group, or channel.

### Private invite link

```bash
./tgsnap https://t.me/+AbCdEf...
./tgsnap https://t.me/joinchat/AbCdEf...
```

Private invites are inspected with Telegram's invite-preview API only.

`tgsnap` does **not**:

- join the group/channel
- send a join request
- fetch message history through invite peek access
- enumerate members beyond preview users Telegram directly returns

If the authenticated account is already a member, the target can be handled as a normal group/channel snapshot.

### Display a saved snapshot

Pass either a snapshot directory:

```bash
./tgsnap --show ~/.local/share/tgsnap/users/123456789_username/snapshots/2026-09-05_00-10-00Z
```

or its `snapshot.json` file:

```bash
./tgsnap --show ~/.local/share/tgsnap/users/123456789_username/snapshots/2026-09-05_00-10-00Z/snapshot.json
```

`--show` is completely local and does not connect to Telegram.

### Override the output directory

```bash
./tgsnap --output /tmp/tgsnap @username
```

### Help

```bash
./tgsnap --help
```

## First Login

The first live lookup starts a Telethon user session. Telegram may ask for your phone number and login code:

```text
Please enter your phone: +15551234567
Please enter the code you received: 12345
```

If two-step verification is enabled, Telethon will also ask for your Telegram password.

After successful authentication, the local session is reused.

## Output

Targets are separated by type:

```text
~/.local/share/tgsnap/
├── users/
│   └── 123456789_username/
│       ├── metadata.json
│       └── snapshots/
├── groups/
│   └── 987654321_groupname/
│       ├── metadata.json
│       └── snapshots/
└── channels/
    └── 456789123_channelname/
        ├── metadata.json
        └── snapshots/
```

Each run creates a new timestamped directory:

```text
123456789_username/
├── metadata.json
└── snapshots/
    ├── 2026-09-04_20-15-22Z/
    │   ├── snapshot.json
    │   └── photos/
    │       ├── profile/
    │       └── fallback/
    └── 2026-09-05_00-10-00Z/
        └── snapshot.json
```

Private invite previews use a short hash-derived storage key instead of writing the private invite hash to disk.

`metadata.json` keeps target-level information such as the current name/username, previously seen usernames, first/last seen timestamps, and the latest snapshot path.

`snapshot.json` contains the point-in-time data collected during that run.

## Snapshot Behavior

Existing snapshots are never updated or deleted.

If a user changes a username, title, bio, photo, or another visible field, the next run records the new state while earlier snapshots remain untouched.

For users, old profile photos are only available if Telegram still exposes them to the authenticated account. A public/fallback profile photo is stored separately from the normal photo history.

Profile music is currently saved as metadata only; the audio files are not downloaded.

## Visibility and Access

`tgsnap` is designed to avoid target-visible interactions.

The current lookup paths do not intentionally:

- join or request to join groups/channels
- send messages
- mark messages as read
- mark stories as seen
- increment channel-post views
- use private-invite peek access to read messages

This does **not** make the requests anonymous from Telegram itself. All live lookups are still performed through your authenticated Telegram account.

`tgsnap` does not bypass Telegram privacy settings or access controls. If Telegram does not expose a field or entity to the authenticated account, `tgsnap` cannot retrieve it.

## Files

```text
.env.example     example configuration
.gitignore       local/session/output exclusions
Pipfile          Pipenv dependencies
Pipfile.lock     locked dependency versions
README.md        project documentation
requirements.txt pip-compatible dependency list
tgsnap           executable CLI
```

## Responsible Use

Use `tgsnap` only for legitimate purposes and only collect information you are permitted to access.

Respect Telegram's [API Terms of Service](https://core.telegram.org/api/terms), users' privacy, and applicable laws and regulations.
