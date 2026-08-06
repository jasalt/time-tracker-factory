# Time-tracker client experiments

A meta-repository for lightweight, native clients for **Clockify** and the
self-hosted **Kimai** time-tracking service. The goal is to explore equivalent
tracker workflows across Go, Rust, and Emacs rather than depend on a large
Electron desktop client.

## Scope

All implementations target the same core workflow: configure/discover an
account, load projects and activities, view the active timer and recent entries,
start/stop/update a timer, continue or delete an entry, and show elapsed and
local-day totals.

| Project        | Focus                                                 | Status                                                          |
|----------------|-------------------------------------------------------|-----------------------------------------------------------------|
| [`gtt`](https://github.com/jasalt/go-time-tracker) | Go core with CLI, Bubble Tea TUI, and Fyne desktop UI | Early development; essential Clockify and Kimai workflows work. |
| [`rtt`](https://github.com/jasalt/rust-time-tracker) | Rust implementation | Planned/experimental. |
| [`ett`](https://github.com/jasalt/emacs-time-tracker) | Native Emacs 30+ status-buffer and Transient UI | Design complete; implementation has not started. |

`gtt` is the current reference implementation. It has provider-neutral core
workflows, Clockify and Kimai adapters, scriptable and interactive frontends,
and a recent live Kimai compatibility run.

## Kimai local testing

The root [`docker-compose.yml`](docker-compose.yml) starts a disposable Kimai
2.x + MySQL test stack. It has been validated with both `docker compose` and
the preferred `podman compose` command.

[`KIMAI-TESTING.md`](KIMAI-TESTING.md) defines implementation-agnostic Kimai
acceptance scenarios for all clients. GTT's captured run, UI screenshots, and
redacted transcript are in the
[`gtt` repository](https://github.com/jasalt/go-time-tracker/tree/master/test-results/2026-08-06T09-34-52Z/).

For Fyne GUI automation and evidence capture, use the direct X11 procedure in
[`docs/VISION-X11.md`](docs/VISION-X11.md).

## Getting started

```sh
git submodule update --init --recursive
podman compose up -d --wait
```

The Compose defaults are disposable-test values only. Read
[`KIMAI-TESTING.md`](KIMAI-TESTING.md) before creating API tokens or using the
stack outside local testing.
