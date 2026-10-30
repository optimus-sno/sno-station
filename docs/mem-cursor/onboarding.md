# Sno Memory for Cursor — Onboarding

This walkthrough describes the first run of `@snoai/mem-cursor`, which gives both the Cursor IDE and
the Cursor CLI the same memory that Claude Code and Codex use through Sno.

## Start setup

Requirements:

- Cursor IDE or Cursor CLI
- Node.js 22.22.3 or newer on the 22 line, 24.15.0 or newer on the 24 line, or 25.9.0 or newer
- `git` on your PATH (memory is scoped to the repository you work in)

Complete the [memory service setup](../memory-setup.md), then install the Cursor integration:

```bash
sno setup --harness cursor
```

The memory service setup writes `~/.sno/settings.json`. Keep a private backup of that file, because it
holds the only key for the encrypted store. The Cursor hooks read `settings.json` from the folder named by
`SNO_PROFILE_DIR` when that variable is set, so if you use it, put the file there.

## What the install changes

In your Cursor folder (`~/.cursor`), `sno setup --harness cursor` writes these four things:

| Where | What it adds |
| --- | --- |
| `hooks.json` | One entry for each of seven Cursor events: `sessionStart`, `beforeSubmitPrompt`, `afterAgentResponse`, `afterAgentThought`, `stop`, `preCompact`, `sessionEnd` |
| `cli-config.json` | The permission rules `Shell(sno)` and `Shell(<path to sno>)`, so the Cursor CLI does not ask before every `sno` command |
| `permissions.json` | The entries `sno` and the path to `sno` in the IDE's terminal allowlist |
| `skills/sno-mem-cursor/SKILL.md` | The skill that teaches the Cursor agent four commands: recall, get, remember, correct |

Entries that other tools wrote into the same files stay where they are. The installer replaces only its
own entries, so running it again to refresh the integration is safe. If one of the files cannot be
parsed, the installer leaves that file as it was, prints which file it skipped, and reports the install
as incomplete; fix the file and run it again.

## Which conversations have memory

- Automatic recall and capture work only inside a git repository. They read and write that repository's
  memory, and recall also searches global memory.
- Outside a git repository, and under the system temporary directory, the hooks read and write nothing.
  The explicit `sno memory` commands still work there, with the folder you run them in as the scope.
- With several folders open, the hooks use the repository of the first folder. To reach another
  repository's memory, run the memory commands from inside that repository's folder.

## Step 1: verify

Open a new Cursor conversation inside a git repository, then run:

```bash
sno memory doctor --harness cursor
```

The output has four lines:

```text
sidecar: healthy
hooks: sessionStart=present beforeSubmitPrompt=present afterAgentResponse=present afterAgentThought=present stop=present preCompact=present sessionEnd=present
permission rule: present
IDE permission rule: present
```

The memory service starts on demand: the first hook that needs it starts it. If `sidecar` shows
`unavailable` before you have opened any conversation, open one in a git repository and run the command
again. `absent` next to a hook means the install did not write that entry; run `sno setup --harness
cursor` again. `unparsable` means that Cursor file could not be read as the JSON the installer expects.

## Step 2: create the first useful memory

Start with information that will help every day: preferred language and tone, project rules,
package-manager rules, preferred tools or commands, and actions the agent should not repeat.

```bash
sno memory remember --harness cursor "Prefer tabs in this repository"
sno memory recall --harness cursor "indentation"
```

Run the commands from inside the repository. You can also ask the Cursor agent to remember something;
the installed skill makes it run the same commands in the terminal. Do not use temporary debug state,
test output, or private infrastructure details as first memories.

## What happens in a conversation

- **At the start**, the conversation receives a short brief of what the repository already established.
  Print mode has no prompt event, so the brief is delivered at the start there.
- **With each message**, memory that matches the prompt is added to what the agent sees. Injected memory is
  marked as data, not instructions.
- **When a turn completes**, the IDE saves the prompt and reply of that turn. Only completed turns with a
  non-empty prompt and reply are saved. The Cursor CLI is different: it reads the conversation transcript
  at each stop and again when the conversation ends.
- **After that**, a background worker stores what was captured. If the end-of-conversation step fails, the
  turns are stored once the conversation has been quiet for 30 minutes.

A hook never blocks Cursor. If one fails or receives input it cannot read, it answers Cursor with an empty
reply and the conversation continues without memory for that step; the cause is written to the hook's
error output.

## Turning parts off

Two settings in `<profile root>/settings.json` control the automatic parts. Explicit
`sno memory remember` and `sno memory recall` keep working either way.

| Setting | Effect when `false` |
| --- | --- |
| `recall.auto` | No brief at the start and no memory added to prompts |
| `capture.ambient` | Turns are not captured (the IDE records each skipped turn as `capture-off`) |

## Onboarding acceptance checklist

Onboarding is complete when:

- `sno setup --harness cursor` finishes without an error;
- `doctor` prints `healthy`, seven hooks `present`, and both permission rules `present` after a new
  conversation;
- the user can create a first memory and find it again with `recall`;
- entries other tools wrote into `hooks.json` are unchanged;
- no secret, private hostname, internal path, or deployment instruction appears in the output.
