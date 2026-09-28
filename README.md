# claude-vscode-remote

Keep working with a Claude Code session that was started from the VS Code extension on a
Remote-SSH server, after the VS Code client that started it is gone.

## The problem

With VS Code Remote-SSH, the Claude Code extension runs on the server, and every Claude
session is a `claude` process whose parent is the **VS Code extension host**. Had you started
`claude` in a terminal, `nohup` or `tmux` would keep it running after the SSH connection
closed. A session started from the extension has no such option: it lives and dies with the
extension host.

Two things go wrong because of that:

- **When the client disconnects**, the VS Code server keeps the extension host running for a
  *reconnection grace time* and then disposes of it. The Claude process goes with it, even in
  the middle of a task. The server's default is 3 hours.
- **When the client process is gone for good** (the laptop restarted, VS Code was quit), the
  new VS Code window cannot reattach to the old extension host: it starts a new one and
  rebuilds its extensions. For stateless extensions that does not matter. Claude Code is
  stateful: the old session is still alive inside the orphaned extension host, the new window
  cannot see it, and it cannot be resumed there because the old process still holds it.

Making the session remote (Remote Control) lets you reach it from claude.ai or the Claude app.
It still cannot be resumed in the new VS Code window while the old extension host is alive.

## The solution

1. **Keep the extension host alive without its client**, by raising the reconnection grace
   time.
2. **Make your sessions remote**, so you can follow and answer them from claude.ai or your
   phone while no VS Code window is connected.
3. **When you connect a new VS Code window**, run `claude-release`: it stops the orphaned
   session process (making the session remote first if it was not), and you resume the
   session in the new window.

### 1. Raise the reconnection grace time

On your local machine (the VS Code client, not the server), add this to your user
`settings.json`:

```json
"remote.SSH.reconnectionGraceTime": 2592000
```

The value is in seconds. 2592000 is 30 days, the maximum.

The client passes this value to the server as `--reconnection-grace-time` when the server
starts, so a server that is already running keeps its old value. To apply the new one, run
**Remote-SSH: Kill VS Code Server on Host...** and reconnect. This kills every Claude session
running under that server, so do it when none of them is working.

To check the value in effect, run this on the server:

```bash
ps -ef | grep -oE -- '--reconnection-grace-time [0-9]+' | sort -u
```

### 2. Make sessions remote

Type `/remote-control` in the Claude Code panel. The session gets a
`https://claude.ai/code/session_...` URL that you can open from a browser or the Claude app.

`claude-release` also does this for any session that is not remote. It only does it when you
release the session, though: if you want to follow a session while you are away, make it
remote before you leave.

### 3. Release the session and resume it in the new window

Open a terminal in the new VS Code window (any SSH shell on the server works too) and list the
live sessions:

```console
$ claude-release
 1834201  4e1c9a02-7b3d-4f5e-9a61-2c8d0b7e5f13  claude-vscode
          title: Refactor the capture pipeline
          remote: https://claude.ai/code/session_01AbCdEfGhIjKlMnOpQrStUv
          started 2026-09-28 09:12 · idle (41 min ago) · last written to the transcript 41 min ago

 1834377  9b0f3e71-25a4-4c8e-b1d6-7e4a3f90c2d8  claude-vscode
          title: Fix the flaky integration test
          started 2026-09-28 09:14 · idle (2 h 5 min ago) · last written to the transcript 2 h 5 min ago

To release one: claude-release <id, id prefix or pid> [--wait | --force]
```

Release one by its session id, a prefix of it, or its pid:

```console
$ claude-release 9b0f
 1834377  9b0f3e71-25a4-4c8e-b1d6-7e4a3f90c2d8  claude-vscode
          title: Fix the flaky integration test
          started 2026-09-28 09:14 · idle (2 h 5 min ago) · last written to the transcript 2 h 5 min ago
Stopped.
It was not remote; making it remote…
Remote: https://claude.ai/code/session_01WxYz0123456789AbCdEfGh
Released. Resume it from the Claude Code panel's session history, or: claude --resume 9b0f3e71-25a4-4c8e-b1d6-7e4a3f90c2d8
```

Then open the session history in the Claude Code panel of the new window and pick the session.
Resuming does not reconnect Remote Control by itself: type `/remote-control` again and the
session reattaches to **the same URL**.

The script refuses to stop the session it is run from, so it will not stop a Claude session
that runs it as a tool.

## Installation

On the server:

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/codeurjc/claude-vscode-remote/main/claude-release \
  -o ~/.local/bin/claude-release
chmod +x ~/.local/bin/claude-release
```

`~/.local/bin` has to be on your `PATH`. Ubuntu adds it at login when the directory exists.

Requirements:

- Linux on the server: the script reads `/proc`.
- Python 3.7 or later, with no extra packages.
- Claude Code with Remote Control available for your account. The script uses the same
  `claude` binary as the session it releases (the one bundled with the VS Code extension), and
  falls back to `claude` on your `PATH`.

## Usage

```text
claude-release                 list live sessions: title, web URL if remote,
                               and what each is doing
claude-release <id|prefix|pid> stop that session's process if it is idle
claude-release <...> --wait    wait until it is idle, then stop it
claude-release <...> --force   stop it even if it is busy
```

A session is **busy** while it is in the middle of a turn. Stopping it then interrupts what it
is doing, so the script refuses unless you pass `--wait` (wait for the turn to end) or
`--force`. An idle session can still own background commands (a long build, a watcher). They
would die with it, so the script lists them and refuses without `--force`.

## How it works

- **Live sessions** come from `~/.claude/sessions/<pid>.json`, where each running Claude Code
  process registers its pid, session id, working directory, status and, while Remote Control
  is on, its remote session id. Titles come from the transcript in `~/.claude/projects/`.
- **Stopping** is `SIGTERM`, so the process flushes its transcript before it exits.
- **Making a session remote** happens right after the stop, because the orphaned process only
  takes requests on its stdin, which belongs to the old extension host. The script resumes the
  session in headless `stream-json` mode, with the same binary and in the same directory as the
  old process. It sends the same `remote_control` control request the VS Code extension sends,
  with `keep_session_on_exit` so the remote session is not archived when that process exits.
  It waits for the connection, prints the URL and closes the process's stdin, which makes the
  process exit. Nothing is sent to the model. The remote session is recorded in the transcript,
  which is why turning Remote Control on after resuming reattaches to the same URL.

## Caveats

- The script relies on Claude Code internals: the session registry and the `stream-json`
  control protocol that the VS Code extension uses. Neither is a public interface. It was tested
  with Claude Code 2.1.283 on Linux, and a later version may break it.
- If making the session remote fails, the script says why and leaves the session released. Once
  you have resumed it, `/remote-control` makes it remote.

## Related

- [anthropics/claude-code#97754](https://github.com/anthropics/claude-code/issues/97754): keep
  the session alive when a Remote Tunnel / Remote-SSH client disconnects, and re-attach the
  panel on reconnect.
- [Remote-SSH release notes, 1.107](https://github.com/microsoft/vscode-docs/blob/main/remote-release-notes/v1_107.md),
  which introduced `remote.SSH.reconnectionGraceTime`.

## License

[Apache 2.0](LICENSE)
