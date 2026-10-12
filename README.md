> **Moved:** herdr-stack now lives in [therealhieu/herdr-plugins](https://github.com/therealhieu/herdr-plugins/tree/main/stack). This repository is archived.
>
> ```sh
> herdr plugin uninstall herdr-stack
> herdr plugin install therealhieu/herdr-plugins/stack
> ```

# herdr-stack

Zellij-style stacked panes for [Herdr](https://herdr.dev), as a plugin on stock Herdr.
The focused pane expands; its siblings collapse to thin strips but stay visible.

```text
┌──────────────┬──────────────┐
│ p1  ▸ strip  │ p4  ▸ strip  │
├──────────────┼──────────────┤
│ p2  focused  │ p5  last     │
│ (fills rest) │ focused      │
├──────────────┤              │
│ p3  ▸ strip  │              │
└──────────────┴──────────────┘
```

- Each side of the tab's top split stacks independently; the top split itself is never resized.
- Works for stacked (`down`) and side-by-side (`right`) splits, nested to any depth.
- Per-tab on/off toggle.
- Uses only Herdr's public socket API (`layout.export`, `pane.layout`, `layout.set_split_ratio`), so it keeps working across Herdr updates.
- A small native binary (Rust, ~0.4 MB, std + `serde_json`) — no runtime dependencies.

## Requirements

- Herdr ≥ 0.9.1 (macOS, Linux, or Windows)
- Rust toolchain (`cargo`, via [rustup](https://rustup.rs)) — the plugin compiles itself on install (~10–40 s)

## Install

From GitHub (runs `cargo build --release --locked` for you):

```sh
herdr plugin install therealhieu/herdr-stack
```

From a local checkout (`plugin link` does not run build commands):

```sh
git clone https://github.com/therealhieu/herdr-stack ~/git/herdr-stack
cd ~/git/herdr-stack && cargo build --release --locked
herdr plugin link .
```

Optional toggle key in `~/.config/herdr/config.toml`:

```toml
[[keys.command]]
key = "prefix+shift+s"
type = "plugin_action"
command = "herdr-stack.toggle"
description = "stack: toggle for this tab"
```

Then `herdr server reload-config`. On Windows use `command = "herdr-stack.toggle-windows"`
(plugin action ids must be unique, so the Windows binary path gets its own action).

## Usage

Split panes as usual and move focus (`prefix+h/j/k/l` or mouse). The focused pane expands.

| Goal | How |
|---|---|
| Toggle stacking for the current tab | `prefix+shift+s` (or `herdr plugin action invoke herdr-stack.toggle`) |
| Disable everywhere | `herdr plugin disable herdr-stack` |
| Re-enable | `herdr plugin enable herdr-stack` |
| Uninstall | `herdr plugin unlink herdr-stack` |

Disabling leaves pane sizes as they are.

## Configuration

Optional `$(herdr plugin config-dir herdr-stack)/config.json`:

```json
{
  "collapsed_rows": 3,
  "collapsed_cols": 20,
  "skip_root": true
}
```

| Key | Default | Meaning |
|---|---|---|
| `collapsed_rows` | `3` | Height of a collapsed pane in a stacked split |
| `collapsed_cols` | `20` | Width of a collapsed pane in a side-by-side split |
| `skip_root` | `true` | Never resize the tab's top split. Set `false` to stack across the whole tab |

## How it works

```text
pane.focused event
  → layout.export          (BSP tree of the focused pane's tab)
  → find path to the focused pane
  → for each split on that path (below the root):
        focused side = total − collapsed size of the other side
  → inside collapsed regions: share space evenly by strip count
  → layout.set_split_ratio for each changed split
```

Runs are serialized with a file lock (`std::fs::File::lock`) so rapid focus changes don't interleave.
Disabled tabs are stored in `$HERDR_PLUGIN_STATE_DIR/disabled-tabs.json`.

## Development

```sh
cargo test                     # stacking plan unit tests
cargo build --release          # herdr picks up the new binary on the next focus event
target/release/herdr-stack <pane_id>   # stack around a pane manually (needs HERDR_SOCKET_PATH)
```

## Limitations

- Herdr clamps split ratios to 0.1–0.9, so a strip is at least 10% of its split. Strips are typically 3–4 rows rather than Zellij's single title line.
- Resizing happens just after the focus change, so a brief re-layout is visible.
- Zoomed tabs are skipped.
- Windows builds compile but have not been tested on a real Windows Herdr yet.

## License

MIT
