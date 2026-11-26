---
name: pallas-editor
description: >-
  Open a file in bjarneo's editor, a native GUI code editor (often installed as
  `editor` or `editor-gui`), and drive it from the agent: point the user at lines
  with a note, or propose a change they accept with y. Use when the user invokes
  /pallas-editor, asks to open a file in editor or editor-gui, asks you to show
  them code in a window, or wants to review a proposed change as a diff. Do not
  use for a pane inside Herdr; use pallas-ttt for that. Do not use when the agent
  only needs to read a file to answer a question.
---

# Open a file in editor

editor (github.com/bjarneo/editor) is a modal code editor in its own native window. It is not a terminal app, so it never runs inside a Herdr pane. For an editor in a pane, use `pallas-ttt`. For a read-only markdown view, use `pallas-glow`.

Written against editor 0.3.2. The `ctl` commands are young; if one fails with a usage error, check `"$ED" ctl --help`.

## Find the command

The command has no fixed name. editor's first-run setup lets the user name it, links `~/.local/bin/<name>` to the binary, and records the name as `cli.name` in `config.toml`. Other programs also answer to `editor`: on Debian and Ubuntu, `/usr/bin/editor` is nano or vim. Trust a candidate only when `--version` prints `editor 0.`.

Set `$ED` to the first candidate that passes, and use `"$ED"` for every command below:

```bash
for cfg in "${XDG_CONFIG_HOME:-$HOME/.config}/editor/config.toml" \
           "$HOME/Library/Application Support/editor/config.toml"; do
  [ -f "$cfg" ] && NAME=$(sed -n '/^\[cli\]/,/^\[/s/^name *= *"\(.*\)".*/\1/p' "$cfg") && [ -n "$NAME" ] && break
done
ED=
for c in ${NAME:+"$NAME" "$HOME/.local/bin/$NAME"} editor-gui "$HOME/.local/bin/editor-gui" editor "$HOME/.local/bin/editor"; do
  "$c" --version 2>/dev/null | grep -q '^editor 0\.' && ED=$(command -v "$c") && break
done
echo "${ED:-not installed}"
```

`config.toml` lives in `~/.config/editor` on Linux and in `~/Library/Application Support/editor` on macOS. The `~/.local/bin` paths cover a Mac where that folder is not on `PATH` yet.

## Install

Only when `$ED` is empty. Ask first. Pick the archive for the system:

| `uname -sm` | Archive |
| --- | --- |
| `Linux x86_64` | `editor-v<ver>-x86_64-linux.tar.gz` |
| `Linux aarch64` | `editor-v<ver>-aarch64-linux.tar.gz` |
| `Darwin arm64` | `editor-v<ver>-aarch64-macos.tar.gz` |
| `Darwin x86_64` | `editor-v<ver>-x86_64-macos.tar.gz` |

Download it and its `.sha256` into a new empty folder, check the sum, and install the binary as `editor-gui`, a name nothing else claims:

```bash
gh release download -R bjarneo/editor -p "$ARCHIVE" -p "$ARCHIVE.sha256"
sha256sum -c "$ARCHIVE.sha256"          # Linux
shasum -a 256 -c "$ARCHIVE.sha256"      # macOS
tar -xzf "$ARCHIVE"
mkdir -p ~/.local/bin && install -m755 editor-v*/editor ~/.local/bin/editor-gui
ED="$HOME/.local/bin/editor-gui"
```

On macOS the binary is not signed. Clear the quarantine flag before the first run:

```bash
xattr -d com.apple.quarantine "$ED" 2>/dev/null || true
```

If the checksum or the install fails, report it and stop. Before the first launch, give the warning in First launch.

## Display

On macOS, skip this step.

On Linux, the window needs the desktop session. Agent shells and Herdr panes often lack it, and editor then panics with `neither WAYLAND_DISPLAY nor WAYLAND_SOCKET nor DISPLAY is set`. When `WAYLAND_DISPLAY` and `DISPLAY` are both empty, take them from the session:

```bash
export XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR:-/run/user/$(id -u)}"
export WAYLAND_DISPLAY="${WAYLAND_DISPLAY:-$(basename "$(ls "$XDG_RUNTIME_DIR"/wayland-[0-9] 2>/dev/null | head -1)")}"
```

Export these in every shell that runs `"$ED"`, including `"$ED" ctl`: the control socket lives under `XDG_RUNTIME_DIR`. If no `wayland-N` socket exists, there is no desktop to draw on (SSH, a headless box). Say so and suggest `pallas-ttt`.

## Target

Use the file or directory the user named. Resolve it to an absolute path. If they did not name one, ask. Add `:LINE` to land on a line. A directory opens as the project. A file opens in its project: the git work tree around it.

## Open

Check for a window on that project first:

```bash
"$ED" ctl list
```

If a listed project contains the target, open it there. This returns at once and brings the window forward:

```bash
"$ED" "$TARGET"
```

If none does, start a window in the background. A first window blocks until it closes, so never run it in the foreground:

```bash
nohup "$ED" "$TARGET" >/dev/null 2>&1 &
```

Then wait until it answers, up to ten seconds:

```bash
for i in $(seq 1 50); do (cd "$PROJECT" && "$ED" ctl state >/dev/null 2>&1) && break; sleep 0.2; done
```

`$PROJECT` is the project folder: the git top level for a file, or the directory itself. Run every `"$ED" ctl` command from that folder, or pass `--socket` with the path `"$ED" ctl list` prints.

### First launch

The first launch ever shows a five-step setup (keys, mouse, look, terminal, ready) that the user finishes in the window. When there is no `config.toml` yet (see Find the command), warn the user before you start the window, in these words or close to them:

> editor's setup has a terminal step that, by default, sets `$EDITOR` and `VISUAL` in your shell startup file and sets git's `core.editor` to editor. With those on, `git commit` without `-m`, `git rebase -i` and anything else that asks for an editor opens a GUI window and waits for it, and fails over SSH. Turn both off in that step unless you want that.

The terminal step also names the command. A name other than `editor-gui` is fine; Find the command reads it from `cli.name`.

If the user already ran setup with those options on and wants them undone:

```bash
git config --global core.editor nvim      # or --unset, or their editor of choice
sed -i.bak "/# Added by editor's setup/,+1d" ~/.bashrc   # ~/.zshrc for zsh
```

Do not change their shell files or git config unless they ask.

Report the path you opened. Do not paste the file into the chat unless the user also asked for an explanation.

## Point at code

To show the user lines, reveal them with a short note. It jumps there, tints the lines and shows the note:

```bash
"$ED" ctl reveal "$FILE:$START-$END" --note "why these lines matter"
"$ED" ctl reveal --clear
```

To see what the user is looking at, read `"$ED" ctl state` (JSON) and `"$ED" ctl selection`. `"$ED" ctl read PATH` returns a buffer with unsaved edits. `"$ED" ctl diagnostics` returns language server errors.

## Propose a change

Prefer `propose`. It shows the new file as a diff in the window. The user presses `y` to accept or `n` to reject, and nothing is saved until they write the buffer:

```bash
"$ED" ctl propose "$FILE" --label "short description" < "$NEW_TEXT_FILE"
```

stdin is the whole new file, not a patch. Build it from `"$ED" ctl read "$FILE"` so unsaved edits survive. Then tell the user a proposal is waiting and what it does. `"$ED" ctl state` shows `"proposals"` until they answer.

Do not use `"$ED" ctl edit`, `"$ED" ctl command` or `"$ED" ctl save` unless the user asked for the change to go in without review.

## Exit status

`"$ED" ctl` exits 0 when done, 1 when the editor refused (the reason is on stderr), and 2 when no editor runs for the folder. On 2, run Open again.
