# Plugin Profiles — User Manual

**Save and restore named plugin combinations for different workflows.**

Version 1.0.0

---

## Table of Contents

- [Quick Start](#quick-start)
- [Command Reference](#command-reference)
  - [`/profile-save`](#profile-save)
  - [`/profile-load`](#profile-load)
  - [`/profile-list`](#profile-list)
  - [`/profile-delete`](#profile-delete)
  - [`/profile-reset`](#profile-reset)
- [Workflows & Recipes](#workflows--recipes)
- [Profile Naming Rules](#profile-naming-rules)
- [Safety Features](#safety-features)
- [Data Storage](#data-storage)
- [Troubleshooting / FAQ](#troubleshooting--faq)

---

## Quick Start

```
/profile-save my-dev-setup      # Save your current plugins as "my-dev-setup"
/profile-list                    # See all saved profiles
/profile-load my-dev-setup       # Restore that set of plugins (restart required)
/profile-reset                   # Enable all installed plugins (clean slate)
```

That's it. You now have a named snapshot of your enabled plugins that you can switch back to at any time.

---

## Command Reference

### `/profile-save`

**Syntax:** `/profile-save <name>`

**What it does:** Reads the currently enabled plugins from `~/.claude/settings.json` and saves them as a named profile. It never modifies settings.json — it only reads from it.

**Step-by-step behavior:**

1. Validates the profile name (see [Naming Rules](#profile-naming-rules)).
2. Reads `enabledPlugins` from `~/.claude/settings.json`.
3. Reads existing profiles from `~/.claude/plugin-data/plugin-profiles/profiles.json` (creates it if missing).
4. If a profile with that name already exists, shows you the existing vs. new plugin lists and asks you to confirm the overwrite.
5. Saves the profile with your enabled plugins, a timestamp, and a plugin count.
6. Confirms the save with the profile name and plugin list.

**Example:**

```
> /profile-save frontend-work

Saved profile 'frontend-work' with 4 plugins (user-level).
- plugin-profiles@local
- ui-ux-pro-max@local
- hookify@local
- context7@registry
```

**Edge cases and errors:**

| Situation | Message |
|---|---|
| Name is empty or invalid | "Invalid profile name. Names must start with a letter, contain only letters/numbers/hyphens/underscores, and be 1-50 characters. Names starting with `_` are reserved." |
| No enabled plugins in settings.json | "No enabled plugins found in settings.json. There's nothing to save. Enable some plugins first." |
| settings.json doesn't exist | Same as above. |
| profiles.json exists but is corrupt JSON | Stops and reports the parse error. |
| Profile name already exists | Shows both plugin lists and asks to confirm overwrite. |

---

### `/profile-load`

**Syntax:** `/profile-load <name>`

**What it does:** Replaces the `enabledPlugins` in `~/.claude/settings.json` with the plugins stored in the named profile. Automatically backs up your current plugins as `_previous` before making changes.

**Step-by-step behavior:**

1. Reads `~/.claude/plugin-data/plugin-profiles/profiles.json`. Stops if no profiles exist.
2. Looks up the requested profile by name. If not found, lists available profiles.
3. Reads your current `enabledPlugins` from settings.json.
4. Reads installed plugins and blocklist to check for plugins that are no longer available.
5. Validates all plugin keys in the profile.
6. Classifies every plugin into one of five categories and shows you a diff:
   - **Will be ENABLED** — in the profile but not currently enabled, and is installed
   - **Will be DISABLED** — currently enabled but not in the profile
   - **Unchanged** — same state in both
   - **Skipped (not installed)** — in the profile but not installed on your system
   - **Skipped (blocked)** — in the profile but on your blocklist
7. Asks you to confirm.
8. Saves your current plugins as the `_previous` auto-backup profile.
9. Writes the new `enabledPlugins` to settings.json (only that key — all other settings are preserved).
10. Reminds you to restart Claude Code.

**Example:**

```
> /profile-load frontend-work

Changes:
  Will be ENABLED:  ui-ux-pro-max@local, context7@registry
  Will be DISABLED: superpowers@local
  Unchanged:        plugin-profiles@local, hookify@local

Apply this profile? This will change your enabled plugins as shown above.
> Yes

Profile 'frontend-work' loaded with 4 plugins.

Auto-backup saved as `_previous` (use `/profile-load _previous` to undo).

RESTART REQUIRED: Run `/exit` and relaunch Claude Code for changes to take effect.
```

**Loading `_previous`:**

```
> /profile-load _previous
```

This works exactly the same way — `_previous` is a valid load target. It restores whatever was enabled before your last load.

**Edge cases and errors:**

| Situation | Message |
|---|---|
| No profiles saved yet | "No profiles saved yet. Use `/profile-save <name>` to create one." |
| Profile not found | "Profile '\<name>' not found. Available profiles: ..." |
| Plugin key fails validation | Warns about the invalid key and skips it. |
| Plugin not installed | Classified as "Skipped (not installed)" in the diff. |
| Plugin is blocked | Classified as "Skipped (blocked)" in the diff. |
| User declines confirmation | No changes made. |
| profiles.json is corrupt | Stops and reports the parse error. |

---

### `/profile-list`

**Syntax:** `/profile-list`

**What it does:** Displays all saved profiles with their plugin counts, timestamps, and plugin names. Takes no arguments.

**Step-by-step behavior:**

1. Reads `~/.claude/plugin-data/plugin-profiles/profiles.json`.
2. For each profile, displays:
   - Profile name (with "(auto-backup)" label if it's an auto-backup like `_previous`)
   - Number of plugins
   - When it was saved
   - List of plugin names
3. Profiles are sorted alphabetically, with `_`-prefixed system profiles shown last.
4. Shows a summary count.

**Example:**

```
> /profile-list

frontend-work — 4 plugins — saved 2026-02-20T14:30:00Z
  plugin-profiles@local, ui-ux-pro-max@local, hookify@local, context7@registry

minimal — 1 plugin — saved 2026-02-18T09:00:00Z
  plugin-profiles@local

_previous (auto-backup) — 3 plugins — saved 2026-02-20T14:32:00Z
  plugin-profiles@local, superpowers@local, hookify@local

3 profile(s) saved. Use `/profile-load <name>` to apply one.
```

**Edge cases and errors:**

| Situation | Message |
|---|---|
| No profiles saved | "No profiles saved yet. Use `/profile-save <name>` to create one." |
| profiles.json doesn't exist | Same as above. |
| profiles.json is corrupt | Stops and reports the parse error. |

---

### `/profile-delete`

**Syntax:** `/profile-delete <name>`

**What it does:** Permanently removes a saved profile. This cannot be undone.

**Step-by-step behavior:**

1. Reads `~/.claude/plugin-data/plugin-profiles/profiles.json`. Stops if no profiles exist.
2. Looks up the profile by name. If not found, lists available profiles.
3. If deleting `_previous` and it's the only remaining profile, gives an extra warning that you're removing your only safety net.
4. Shows the profile's details (plugin count, timestamp, plugins, auto-backup status) and asks to confirm deletion.
5. Removes the profile entry from profiles.json and writes the file back.
6. Confirms deletion with the count of remaining profiles.

**Example:**

```
> /profile-delete minimal

Profile 'minimal':
  1 plugin — saved 2026-02-18T09:00:00Z
  plugin-profiles@local

Delete profile 'minimal'? This cannot be undone.
> Yes

Deleted profile 'minimal'. 2 profile(s) remaining.
```

**Edge cases and errors:**

| Situation | Message |
|---|---|
| No profiles saved | "No profiles saved yet. Nothing to delete." |
| Profile not found | "Profile '\<name>' not found. Available profiles: ..." |
| Deleting `_previous` when it's the only profile | Extra warning: "The `_previous` profile is your only safety net for undoing profile loads. Are you sure you want to delete it?" |
| User declines confirmation | No changes made. |
| profiles.json is corrupt | Stops and reports the parse error. |

---

### `/profile-reset`

**Syntax:** `/profile-reset`

**What it does:** Resets `enabledPlugins` in `~/.claude/settings.json` to include all installed plugins minus any blocked plugins. This gives you a clean slate with everything enabled. Takes no arguments.

**Step-by-step behavior:**

1. Reads `~/.claude/plugins/installed_plugins.json` to get all installed plugins. Stops if the file doesn't exist.
2. Reads `~/.claude/plugins/blocklist.json` to check for blocked plugins (if the file doesn't exist, no plugins are excluded).
3. Reads `~/.claude/settings.json` to get the current `enabledPlugins`. Stops if the file doesn't exist.
4. Builds the reset state: all installed plugins set to `true`, minus blocked plugins. Always includes `plugin-profiles@local` (self-preservation).
5. Validates all plugin keys against the expected format. Warns about and skips any invalid keys.
6. Compares the current state to the reset state and shows a diff with categories: **Will be ENABLED**, **Will be DISABLED**, **Unchanged**, **Skipped (blocked)**.
7. If the current state already matches the reset state, reports that there's nothing to change and stops.
8. Asks for confirmation. If confirmed, saves the current plugins as `_previous` (auto-backup) and writes the new `enabledPlugins` to settings.json.

**Example:**

```
> /profile-reset

Changes:
  Will be ENABLED:  superpowers@local, context7@registry
  Will be DISABLED: (none)
  Unchanged:        plugin-profiles@local, hookify@local
  Skipped (blocked): sketchy-plugin@local

Reset enabled plugins to all installed plugins as shown above?
> Yes

Reset to 4 plugins (all installed minus blocked).

Auto-backup saved as `_previous` (use `/profile-load _previous` to undo).

RESTART REQUIRED: Run `/exit` and relaunch Claude Code for changes to take effect.
```

**Edge cases and errors:**

| Situation | Message |
|---|---|
| `installed_plugins.json` doesn't exist | "Cannot reset: `installed_plugins.json` not found. Claude Code needs at least one installed plugin to reset to." |
| `settings.json` doesn't exist | "Cannot reset: `settings.json` not found." |
| Current state already matches reset state | "Your enabled plugins already match the full installed set. Nothing to reset." |
| Blocked plugins exist | Classified as "Skipped (blocked)" in the diff. |
| User declines confirmation | No changes made. |

---

## Workflows & Recipes

### Save and switch between workflows

```
# Set up plugins for frontend dev, save it
/profile-save frontend-work

# Reconfigure plugins for data analysis, save that too
/profile-save data-analysis

# Switch between them whenever you need to
/profile-load frontend-work
# restart Claude Code
/profile-load data-analysis
# restart Claude Code
```

### Undo a profile load with `_previous`

Every time you run `/profile-load`, your current plugins are automatically backed up as `_previous`. To undo:

```
/profile-load _previous
# restart Claude Code
```

This restores whatever was enabled immediately before the last load.

### Overwrite an existing profile

Just save with the same name. You'll see the existing profile's details alongside the new one and be asked to confirm:

```
/profile-save frontend-work

Profile 'frontend-work' already exists:
  Saved: 2026-02-19T10:00:00Z — 3 plugins
  Existing: plugin-profiles@local, hookify@local, context7@registry
  New:      plugin-profiles@local, hookify@local, context7@registry, ui-ux-pro-max@local

Overwrite?
> Yes

Saved profile 'frontend-work' with 4 plugins (user-level).
```

### Clean up old profiles

List profiles to see what you have, then delete the ones you don't need:

```
/profile-list
/profile-delete old-experiment
/profile-delete another-old-one
```

### Start fresh with all plugins

If you want a clean slate with every installed plugin enabled:

```
/profile-reset
# restart Claude Code
```

This enables all installed plugins minus blocked ones — no need to have a saved profile.

### Start fresh after experimenting

If you've been testing plugin combinations and want to get back to a known-good state:

```
/profile-load my-stable-setup
# restart Claude Code
```

Or if you saved before experimenting:

```
/profile-load _previous
# restart Claude Code
```

---

## Profile Naming Rules

| Rule | Detail |
|---|---|
| **Allowed characters** | Letters (`a-z`, `A-Z`), numbers (`0-9`), hyphens (`-`), underscores (`_`) |
| **Must start with** | A letter |
| **Length** | 1 to 50 characters |
| **Case sensitive** | Yes — `MyProfile` and `myprofile` are different profiles |
| **Reserved prefix** | Names starting with `_` are reserved for system use (e.g., `_previous`) |

**Regex:** `^[a-zA-Z][a-zA-Z0-9_-]{0,49}$`

**Valid examples:** `dev`, `frontend-work`, `my_setup_v2`, `DataAnalysis`, `a`

**Invalid examples:** `_private` (reserved prefix), `2fast` (starts with number), `-setup` (starts with hyphen), `my profile` (contains space), `a-very-long-name-that-exceeds-the-fifty-character-maximum-limit` (too long)

---

## Safety Features

### Auto-backup (`_previous`)

Every time you load a profile or reset, your current plugin configuration is automatically saved as `_previous` before any changes are made. This means you can always undo the last load or reset by running `/profile-load _previous`. The `_previous` profile is overwritten each time you load or reset, so it always reflects the state immediately before the most recent change.

### Self-preservation

When loading a profile or resetting, `plugin-profiles@local` is always included in the resulting `enabledPlugins`, even if the saved profile didn't contain it. This ensures the plugin-profiles plugin stays functional after every load or reset — you can never accidentally lock yourself out of your profiles.

### Read-only save

The `/profile-save` command never writes to `settings.json`. It only reads from it. Your active plugin configuration is never modified by saving a profile.

### Uninstalled and blocked plugin filtering

When loading a profile, each plugin is checked against your installed plugins list (`~/.claude/plugins/installed_plugins.json`) and blocklist (`~/.claude/plugins/blocklist.json`). Plugins that are not installed or are blocked are skipped and reported in the diff — they won't be enabled. This prevents errors from trying to enable plugins that aren't available on your system.

If either of these files doesn't exist on your system, that check is simply skipped — load will not fail because of a missing installed_plugins.json or blocklist.json.

Additionally, any plugin key in a profile that doesn't match the expected format (`name@source`) is silently skipped with a warning during load.

### Restart requirement

Changes to `enabledPlugins` in settings.json only take effect when Claude Code starts up. After loading a profile, you must exit and relaunch. The load command always reminds you of this.

---

## Data Storage

### Location

All profiles are stored in a single file:

```
~/.claude/plugin-data/plugin-profiles/profiles.json
```

### Schema

```json
{
  "version": 1,
  "profiles": {
    "my-profile": {
      "enabledPlugins": {
        "plugin-profiles@local": true,
        "hookify@local": true
      },
      "savedAt": "2026-02-20T14:30:00.000Z",
      "pluginCount": 2
    },
    "_previous": {
      "enabledPlugins": {
        "plugin-profiles@local": true,
        "superpowers@local": true
      },
      "savedAt": "2026-02-20T14:32:00.000Z",
      "pluginCount": 2,
      "autoBackup": true
    }
  }
}
```

**Fields:**

| Field | Type | Description |
|---|---|---|
| `version` | number | Schema version (currently `1`) |
| `profiles` | object | Map of profile name to profile data |
| `profiles.<name>.enabledPlugins` | object | Map of plugin IDs to `true` — mirrors the format in settings.json |
| `profiles.<name>.savedAt` | string | ISO 8601 timestamp of when the profile was saved |
| `profiles.<name>.pluginCount` | number | Number of plugins in the profile |
| `profiles.<name>.autoBackup` | boolean | (Optional) `true` for auto-generated backups like `_previous` |

### Inspecting manually

You can view profiles.json directly with any text editor or JSON viewer. It's standard JSON and safe to read. However, editing it by hand is not recommended (see FAQ below).

---

## Troubleshooting / FAQ

### "Why do I need to restart after loading a profile?"

Claude Code reads `settings.json` at startup to determine which plugins to enable. Changes made to the file during a session don't take effect until the next launch. Run `/exit` and relaunch Claude Code after loading a profile.

### "How do I undo a load?"

Run `/profile-load _previous`. This loads the auto-backup that was created right before the last load. Then restart Claude Code.

### "What happens to plugins not in the profile?"

They get disabled. Loading a profile replaces the entire `enabledPlugins` object — it's not a merge. If a plugin isn't in the profile, it won't be in the new `enabledPlugins` (with the exception of `plugin-profiles@local`, which is always preserved).

### "Can I edit profiles.json by hand?"

Technically yes, since it's plain JSON. However, the commands validate plugin keys against the pattern `^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+$` and profile names against their own pattern. If you introduce values that don't match these patterns, the commands may warn or skip them. If you corrupt the JSON structure, all commands will refuse to operate and report the parse error.

### "What if profiles.json gets corrupted?"

All five commands check for valid JSON before operating. If the file can't be parsed, they stop immediately and report the error — they will never guess or reconstruct data. To recover, you can either fix the JSON manually or delete the file entirely (you'll lose all saved profiles, but the commands will treat it as a fresh start).

### "What if I save a profile and then uninstall one of its plugins?"

When you load that profile later, the uninstalled plugin will appear as "Skipped (not installed)" in the diff summary. It won't be enabled. All other plugins in the profile will load normally.

### "Can I load a profile that doesn't include plugin-profiles?"

Yes. The self-preservation feature automatically adds `plugin-profiles@local` to the loaded plugins, so you'll never lose access to your profiles.

### "What does profile-reset do differently from profile-load?"

`/profile-reset` enables **all** installed plugins (minus blocked ones) — it doesn't use a saved profile. `/profile-load` restores a specific saved set of plugins. Use reset when you want everything enabled; use load when you want a curated subset.

### "Does saving a profile capture any settings besides plugins?"

No. Profiles only capture the `enabledPlugins` object. Other settings in `settings.json` (like `skipDangerousModePermissionPrompt`, API keys, etc.) are never read, saved, or modified by this plugin.
