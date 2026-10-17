# fast-jev-compaction

A Claude Code plugin that replaces the built-in `/compact` summary with Jev
decisions. Every tool call and tool result in the conversation is scored by
TypeSafe's Jev model (served through OpenRouter); the ones no longer needed are
dropped or truncated, and everything else stays verbatim. User and assistant
text is never rewritten.

This fork sends Jev requests to OpenRouter
(`https://openrouter.ai/api/v1/systemone`, model `typesafe/jev-1.13`) instead
of TypeSafe's own endpoint.

Our organization manages Claude Code, so the plugin is deployed as a
**managed plugin**: an admin lists it once in the organization's managed
settings, and each member puts a copy of this repository at a fixed path on
their machine. A copy installed with `/plugin install` from GitHub does **not**
work here: on a managed machine Claude Code skips `session.compact` hooks from
plugins people install themselves, and `/compact` silently uses the built-in
summary.

## Install

### Admin (once for the organization)

Done by an Owner or Primary Owner.

1. Open claude.ai → **Organization settings** → **Claude Code** →
   **Managed settings**.
2. Add these keys to the managed settings, keeping any keys already there:

   ```json
   {
     "extraKnownMarketplaces": {
       "fast-jev-compaction": {
         "source": { "source": "directory", "path": "/opt/claude-plugins/fast-jev-compaction" }
       }
     },
     "enabledPlugins": { "fast-jev-compaction@fast-jev-compaction": true },
     "appendPlugins": ["fast-jev-compaction@fast-jev-compaction"]
   }
   ```

   The marketplace key must be exactly `fast-jev-compaction`, the `name` in
   `.claude-plugin/marketplace.json`. Any other name makes Claude Code skip
   the plugin.

### Each team member

1. **Put the plugin at the managed path.** Run this in a normal terminal, not
   with `!` inside Claude Code, since `sudo` needs to ask for your password:

   ```sh
   sudo mkdir -p /opt/claude-plugins
   sudo git clone https://github.com/mardawi/fast-jev-compaction /opt/claude-plugins/fast-jev-compaction
   ```

   The folder must be owned by root so only an admin can change it.

2. **Remove any personal install.** If you installed the plugin from GitHub
   before, remove it inside Claude Code:

   ```
   /plugin uninstall fast-jev-compaction@fast-jev-compaction
   /plugin marketplace remove fast-jev-compaction
   ```

   Then delete any `fast-jev-compaction` entries under `enabledPlugins` and
   `extraKnownMarketplaces` in `~/.claude/settings.json`.

3. **Turn on function hooks** by adding this to `~/.claude/settings.json`:

   ```json
   { "env": { "CLAUDE_CODE_ENABLE_FUNCTION_HOOKS": "1" } }
   ```

4. **Restart Claude Code.** Quit every session and start `claude` again. The
   plugin installs and enables itself from the managed settings; there is
   nothing to install by hand. `/reload-plugins` is not enough the first time.

5. **Add your OpenRouter API key.** Open `/plugin`, select
   **fast-jev-compaction**, choose **Configure** and paste your key into
   **TypeSafe API key**. It is stored as a secret, not in a settings file.
   The key must be an **OpenRouter** key: the plugin sends its requests to
   OpenRouter. (The setting is still called "TypeSafe API key" for
   historical reasons.)

   You can instead set it as an environment variable in
   `~/.claude/settings.json`, but that stores the key in plain text:

   ```json
   { "env": { "TYPESAFE_API_KEY": "<your OpenRouter key>" } }
   ```

6. **Check that it works** (see below).

### Check that it works

1. Start Claude Code with debug logging: `claude --debug`.
2. Work for a while, so the conversation has a good number of tool calls
   (file reads, commands), then run `/compact`.
3. Watch for the toast:
   - `kept N/M messages, no summary (…)`: the plugin compacted the
     conversation.
   - `fallback to built-in summary (below 25% minimum: …)`: the plugin ran,
     but dropping the stale tool calls would have saved less than
     `minReductionRatio`, so it let the built-in summary run. This is normal
     in short sessions.
   - `fallback to built-in summary (<error>)`: the plugin ran and failed. The
     reason is in the brackets.
   - No toast at all: the plugin did not run. See Troubleshooting.
4. Look at the newest log, `~/.claude/debug/latest`:

   ```sh
   grep -n "fast-jev-compaction" ~/.claude/debug/latest | grep -v "IdeaProjects\|github.com"
   ```

   A working install shows:

   ```
   hooks module fast-jev-compaction@fast-jev-compaction loaded (worker, …, tier append); events: session.compact,turn.complete
   $.http.fetch (fast-jev-compaction): POST https://openrouter.ai/api/v1/systemone
   $.http.fetch (fast-jev-compaction): 200 in …
   ```

### Updating

Pull the new version into the managed folder, then restart Claude Code:

```sh
sudo git -C /opt/claude-plugins/fast-jev-compaction pull
```

`/plugin update` does not apply to a managed plugin loaded from a folder.

## Configuration

Set options per person in `/plugin` → **fast-jev-compaction** →
**Configure**. To set options for everyone, add them to the managed settings
under `pluginConfigs` (never put `apiKey` there, since every member would get
the same key):

```json
{
  "pluginConfigs": {
    "fast-jev-compaction@fast-jev-compaction": {
      "options": { "minReductionRatio": 0.15, "compactAtPercent": 70 }
    }
  }
}
```

| Option | Setting name in `/plugin` | Default | What it does |
| --- | --- | --- | --- |
| `apiKey` | TypeSafe API key | unset | Your OpenRouter key. When unset, the plugin reads `TYPESAFE_API_KEY` from the environment, then from `env` in your settings. |
| `keepThreshold` | Keep threshold | `0.5` | Minimum Jev probability (0–1) for a tool call or result to stay. Raise it to drop more. |
| `preserveRecentMessages` | Recent messages to preserve | `6` | Number of newest messages never touched. The first message is always kept too. |
| `compactAtPercent` | Compaction percentage | `60` | Context usage (%) at which the plugin starts a compaction at the end of a turn. |
| `minReductionRatio` | Minimum reduction ratio | `0.25` | Minimum share of the conversation (0–1) the plugin must remove to replace the history; below it the built-in summary runs instead. |
| `maxStateTokens` | Maximum Jev state tokens | `25000` | Token budget for the conversation copy sent with every Jev request. Older parts are shortened to fit. |
| `maxRequestTokens` | Maximum Jev request tokens | `30000` | Token budget for one request (conversation plus questions); larger jobs are split into several requests. Keep it under Jev's 32k limit. |
| `truncateHeadChars` | Truncated tool result head | `300` | Characters kept from a tool result whose full output is no longer needed. `0` keeps only a one-line note. |
| `model` | Jev model | `jev-latest` | **Currently ignored.** This fork always uses `typesafe/jev-1.13` via OpenRouter. |

### How a compaction is decided

1. Each tool call is paired with its result. Calls in the first message and in
   the newest `preserveRecentMessages` messages are pinned and never touched.
2. Jev sees the whole conversation (tool outputs replaced by a short note) and
   answers two questions per call: is the call still needed, and is its full
   result still needed.
3. Per call, against `keepThreshold`:
   - result score ≥ threshold: keep the call and its result;
   - else call score ≥ threshold: keep the call, cut the result to
     `truncateHeadChars` characters plus a note;
   - else: remove the call and its result.
4. If the result removes less than `minReductionRatio` of the conversation,
   or anything fails (missing key, request error, bad answer), the plugin
   hands over to Claude Code's built-in summary.

Each compaction logs a `decisions:` line with every call's scores, visible in
the debug log.

## Troubleshooting

Read `~/.claude/debug/latest` after starting Claude Code with `claude --debug`.

| Log line or symptom | Cause | Fix |
| --- | --- | --- |
| `session.compact bypassed by cc-plugin-sec-default (tier user)` | The plugin was installed from GitHub with `/plugin install`, so it counts as a personal plugin. | Remove the personal install (step 2) and use the managed install. |
| `marketplace at the declared path names itself "fast-jev-compaction"` | The marketplace key in managed settings is not `fast-jev-compaction`. | Rename the key and both plugin ids in managed settings to `fast-jev-compaction`. |
| `Skipping orphaned enabledPlugins entry …` or `appendPlugins names … skipped` | Same name mismatch, or `/opt/claude-plugins/fast-jev-compaction` does not exist. | Check the managed settings and the folder. |
| `Remote settings: Cache still valid (304)` and the plugin is missing | Claude Code is still using the settings it cached earlier. | Quit every session and start Claude Code again. |
| Toast `fallback to built-in summary (TYPESAFE_API_KEY is not configured)` | No key found. | Set the key (step 5). |
| Toast `fallback to built-in summary (Jev request failed (401): …)` | The key is not a valid OpenRouter key. | Use an OpenRouter key. |
| The plugin is missing from `/plugin` | Claude Code has not loaded it; the reason is in the log. | Restart Claude Code and check the log lines above. |

## Development

```sh
npm install
npm run typecheck        # library + hook
npm test
npm run validate:plugin  # claude plugin validate
```

To test a change to the hook, push it, pull it into
`/opt/claude-plugins/fast-jev-compaction` and restart Claude Code.

The library in `src/` is also usable on its own; see the upstream repository,
[tamaratran/fast-jev-compaction](https://github.com/tamaratran/fast-jev-compaction),
for the library documentation.
