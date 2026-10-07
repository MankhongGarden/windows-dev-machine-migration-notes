# Windows dev machine migration notes

> **Status (2026-10-07): built and rehearsed, but not yet used for a real move.**
> The weekly backup is running. The restore was rehearsed three times inside a throw-away Windows user on the same laptop. The last rehearsal finished with 0 failed steps.
> Four parts have never run on real new hardware: partitioning the disk, installing apps, importing registry settings, and registering admin scheduled tasks.
> This page describes what was built and why. It is not a tested recipe. I will update it after the first real migration.

## The goal

I wanted a new PC to come back like an iPhone does: sign in, wait, and everything is where it was.

My laptop is more than a normal Windows setup:

- an AI coding agent (Claude Code) does most of the development, with its config, skills, memory and full chat history
- about 110 environment variables, most of them API keys and tokens
- around 15 git repos that live outside any cloud folder, some with commits that were never pushed
- scheduled tasks, tray tools and registry tweaks that keep the machine cool and the C: drive small

Windows' own backup brings back none of that. I am not buying a new PC yet. So the plan became a **permanent weekly backup plus one restore script**. That way the day a laptop dies is just a normal restore, not an emergency.

## What it does

### Backup (every Sunday, in the background)

- **One zip per part, stored on personal OneDrive.** The parts are:
  - tools
  - Claude Code config and history
  - local folders that are not in any cloud
  - git
  - system state

  It keeps the last 4 runs, about 1 GB each. One run takes about 15 minutes. The zips are set to online-only, so they don't take local disk space.
- **Git:** `git bundle --all` for every repo outside a cloud folder, plus the uncommitted changes. This captures unpushed work without pushing anything. Some of these repos deploy automatically on push, so pushing from a backup job would be dangerous.
- **AI agent state:** settings, skills, memory and conversation transcripts. Login credential files are left out on purpose, so you sign in again on the new machine.
- **System state:**
  - scheduled task XML
  - selected registry keys
  - junctions
  - the winget and Store app lists
  - npm and pip globals
  - power plan
  - fonts
  - Windows Terminal settings
- **Status everywhere I already look:** a line in a tray icon, plus a mention in my morning chat summary if the backup fails or is more than 10 days old.

### Secrets never leave the machine in plain text

- Every user environment variable and every SSH key is synced to **Bitwarden Secrets Manager** (free plan). The backup job signs in with a machine-account token that is encrypted with Windows DPAPI, so it only works for this Windows user.
- Before zipping, every staged text file is searched for the real secret values. Each one found is replaced with a placeholder like `<<BW:STRIPE_SECRET_KEY>>`. On restore, the values are fetched from Bitwarden and put back.
- **The trap I fell into:** a secret that was never an environment variable is invisible to that step. API keys pasted into chats with the agent end up in its prompt history and transcripts. So there is now a second pass: a **pattern scan** (GitHub, Stripe, Anthropic, OpenAI, Google, Meta, Supabase, private keys, URLs with a password in them, and more). It reports only a fingerprint of each match, never the value. Once approved, a value is stored in Bitwarden as an extra rule and is hidden from the next backup on.
  - The first scan of a real backup found **37 such values (28 high-risk)**.
- I review the scan results as part of a regular machine health check that the agent runs.

### Restore (one double-click)

`bootstrap.cmd` installs PowerShell 7 if it is missing, then runs `restore.ps1` in steps. Each step can be run again on its own:

1. **Partition:** ask for the C: size, then split the disk to recreate D:, because many paths depend on it.
2. **Fetch:** download the backup and ask once for a new Bitwarden access token.
3. **Apps:** build a checklist from the winget list. Apps I opened in the last 90 days are pre-ticked, using Windows' own app-usage record. It then installs everything ticked with one `winget configure` file.
4. **Files:** put every folder back at its original path. Paths under the old `C:\Users\<name>` are remapped to the new profile. Secret placeholders are filled back in.
5. **Git:** clone every repo from its bundle. If a bundle can't be cloned, it falls back to cloning from the remote, then applies the uncommitted changes.
6. **Env:** set the environment variables and SSH keys from Bitwarden.
7. **Registry, junctions, scheduled tasks:** the steps that need admin rights run in a separate script behind one UAC prompt.
8. **Metrics:** a small logger records temperature, clock speed, RAM and disk every 5 minutes. I can then decide on the new machine's CPU cap and power settings from data instead of guessing.
9. **Checklist:** a list of what can't be moved (sign-ins, OAuth consents) for me to finish by hand.

Wallpaper, theme and Start menu pins come back through Windows' own **"Restore from this PC"** when you set up Windows. The plan is to use both: Windows Backup for the look, this kit for everything else.

### Rehearsal

`rehearse.ps1` creates a temporary local Windows user, runs the restore as that user into a test folder, checks the result from inside that user, and deletes the user afterwards. In the final run:

- 110 environment variables restored
- the SSH key restored
- 15 repos cloned
- the full agent config came back: 66 skills and 476 transcripts
- 9 scheduled tasks registered (5 more were listed as needing admin)

## Lessons so far

- **`Start-Process -Credential` passes on the caller's environment variables.** Without `-UseNewEnvironment`, my "other user" rehearsal quietly wrote into my own profile. Inside restore code, use the user identity and `GetFolderPath` rather than `$env:USERPROFILE`.
- **"I turned the computer off" can mean sleep.** A backup that started before sleep picked up again after wake and collided with the next scheduled run. A named mutex fixed it.
- **My first backup contained plaintext secrets.** Secret redaction was added only after I looked at what was actually inside the zips. Check your backups for secrets before you trust them.
- **Big strings in PowerShell regex can be very slow.** The first version of the scan ran for more than 20 minutes on one backup before I stopped it. On the ~1 GB of transcripts, moving the regex into an `Add-Type` C# method while still reading the files in PowerShell took about 2 minutes. Reading the files *inside* C# took 17 seconds. Also avoid patterns that start with an alternation or a character class (`(?:sk|rk)_` → `[sr]k_`), because that turns off the fast literal search.
- **Shallow clones can't be re-cloned from a bundle.** You need a fallback to the remote.

## What I looked at first

Before building this I looked at existing approaches:

- commercial PC-to-PC transfer tools
- dotfile managers like chezmoi
- `winget configure` (DSC)
- a few scripts for syncing Claude Code config

The kit uses `winget configure` for apps. I didn't find an existing setup that also handled the secrets, the unpushed git work and the AI agent's state in one pass, so the rest is custom. This was a quick search, not a full survey.

## What's next

- The first real migration, after which this page gets a results section with what broke.
- Until then, the code stays private. It is full of paths and names that only make sense on this one laptop.

---

*Written with Claude Code, the agent that also built and runs the kit.*
