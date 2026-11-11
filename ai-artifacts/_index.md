# ai-artifacts

How-the-project-works docs for agents live here. Update them when behavior or architecture changes.

Human-facing sources already in the tree:

- `README.md`
- `docs/` (`docs/publishing.md`)
- `scripts/` (`scripts/install-dependencies.sh`)

Deeper agent docs (context glossaries, ADRs, architecture notes) should grow in this folder when there is a real decision or term to record.

## Clipboard

`copy_to_clipboard` tries `pbcopy` on macOS, `wl-copy` when `WAYLAND_DISPLAY` is set, then `xclip -selection clipboard` and `xsel --clipboard --input` when `DISPLAY` is set. If those fail, or when there is no display, it writes OSC 52 (`ESC ] 52 ; c ; <base64> BEL`) to `/dev/tty`. A set `TMUX` variable wraps that write in a tmux DCS passthrough. Base64 longer than 100 KB is refused. The function returns `"copied"` when a native tool exits 0 and `"sent"` after an OSC 52 write, which the terminal cannot confirm. Ctrl-Y shows "Link copied to clipboard", "Link sent to terminal clipboard", or "Could not copy link to clipboard". Opening a file still proceeds when the path copy fails.

## Pane capture (Herdr)

`HerdrBackend.capture_pane_text` reads every pane in the source pane's tab. `_read_visible` uses `herdr pane read --source recent-unwrapped --lines <viewport_rows>`, which is the visible screen with terminal soft wraps joined; a pane scrolled into history, or a failed read, falls back to `--source visible`.

Agents that draw their own TUI (Claude Code, pi) hard-wrap long lines with a newline and an indent, which no terminal-level unwrap can undo. `_read_pane` therefore looks up the pane's `agent_session` from `herdr pane list`. `agent_transcript_path` maps a Claude session id to `$CLAUDE_CONFIG_DIR/projects/*/<id>.jsonl` (default `~/.claude`), a pi session id to `$PI_CODING_AGENT_DIR/sessions/*/*_<id>.jsonl` (default `~/.pi/agent`), or uses a `kind = "path"` value directly. `read_transcript_text` collects every string from the last 4 MB of that JSONL. `rejoin_agent_wraps` then joins a row to the next when the last token of one (leading brackets/quotes stripped) plus the first token of the next (trailing punctuation stripped) is at least 12 characters, looks like a URL or path (`/`, `\`, `://` or a file extension), and occurs verbatim in the transcript. Joins chain across several rows. Link and path extraction run unchanged on the result. `TERMSCOPE_AGENT_TRANSCRIPTS=0` turns this off. The tmux backend has no session mapping and does not do this.

## Issue and PR refs (link picker)

`cmd_links` passes `extract_visible_links` a resolver from `forge_ref_resolver`. `extract_forge_refs` finds `#N`, `repo#N` and `owner/repo#N` outside URLs (URLs are blanked first, so fragments never match; a ref after a word character, `/`, `.`, `-`, `&` or `#` is skipped). On the first ref, the resolver looks up the source pane's directory (`resolve_source_pane_path`) and its `origin` remote through `forge_remote`: `parse_remote_url` handles https, scp-style and `ssh://` remotes, owner being everything before the last path segment; an SSH host goes through `_web_host` (`ssh -G` hostname, then `getaddrinfo` canonical name for a dotless name). Refs become `https://<host>/<owner>/<repo>/issues/N` and join the URL list on their screen line. Screens without a ref never run git. No remote drops the refs. `TERMSCOPE_FORGE_REFS=0` turns this off. `Ctrl-E` in the link picker passes the selected URL to `open_in_nvim_split` as is.
