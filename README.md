# herdr-equalize-splits

A tiny [herdr](https://herdr.dev) plugin that balances **every split in the current tab** so each pane gets an equal share of screen area. Think tmux's `select-layout` even, or Ghostty's `equalize_splits`, but scoped to herdr's BSP tree.

Bind it to `prefix+=` (i.e. `Ctrl+b =`) and one keystroke flattens whatever lopsided mess you've resized your panes into.

```
before                         after  (prefix+=)
┌────────────┬──┬─────┐        ┌──────┬──────┬──────┐
│            │  │     │        │      │      │      │
│     p1     │p2│ p3  │   ->   │  p1  │  p2  │  p3  │
│            │  │     │        │      │      │      │
└────────────┴──┴─────┘        └──────┴──────┴──────┘
   70%        7%  22%             33%    33%    33%
```

## What "equalize" means here

herdr models a tab as a **binary** BSP tree. Each `split` has a direction
(`right` = side-by-side columns, `down` = stacked rows) and a `ratio` (the share
given to its first child). There is no three-way container — three columns are
really `p1 | (p2 | p3)`.

This plugin walks that tree and, for every split, sets

```
ratio = leaves(first) / (leaves(first) + leaves(second))
```

Weighting each divider by the number of panes on each side is what makes the
result **equal area per pane**, not just "every divider at 50%":

| layout                         | result                                  |
|--------------------------------|-----------------------------------------|
| 3 columns `p1 \| p2 \| p3`     | 33% / 33% / 33% width                    |
| 2 rows `p1 / p2`               | 50% top / 50% bottom                     |
| mixed `p1 \| (p2 \| (p3 / p4))`| ~25% area each                           |

It never splits, closes, or recreates panes — it only moves dividers via the
`layout.set_split_ratio` socket call, so running processes are untouched.

## Requirements

- herdr **0.7.0+** (uses the `layout.export` / `layout.set_split_ratio` socket API)
- `node` on `PATH` (the action runs `node equalize.mjs`)

## Install

```sh
# from a local clone
herdr plugin link /path/to/herdr-equalize-splits

# verify it registered
herdr plugin action list --plugin local.equalize-splits
```

## Keybinding

Add to `~/.config/herdr/config.toml`, then `herdr server reload-config`:

```toml
[[keys.command]]
key = "prefix+="
type = "plugin_action"
command = "local.equalize-splits.equalize"
description = "equalize splits"
```

`prefix` is `Ctrl+b` by default, so this binds `Ctrl+b =`. A ready-to-copy
version lives in [`config.example.toml`](./config.example.toml).

> Editing `herdr-plugin.toml` (adding/renaming actions)? Re-run
> `herdr plugin link <path>` to re-register the action list —
> `herdr server reload-config` only re-reads keybindings.

## Actions

| action id            | what it does                                    |
|----------------------|-------------------------------------------------|
| `equalize`           | balance every split in the focused tab          |
| `equalize-dry-run`   | print the planned ratios without changing them  |

Run either from the CLI too:

```sh
herdr plugin action invoke equalize --plugin local.equalize-splits
```

## How it works (internals)

1. `layout.export` returns the focused tab's tree (`{split{direction,ratio,first,second}|pane}`).
2. `equalize.mjs` computes a target ratio + boolean tree path for each split
   (`false` = descend into `first`, `true` = `second`).
3. Each `(path, ratio)` is pushed back with `layout.set_split_ratio`.

Everything talks to the server over `HERDR_SOCKET_PATH` with newline-framed JSON
requests (`{id, method, params}` → `{id, result}`).

## License

MIT
