# Kimai integration testing

This document defines the provider-facing acceptance contract for a time-tracking
client. It deliberately does not prescribe a particular client implementation,
programming language, command syntax, or UI toolkit. It applies to `gtt` today
and is intended to be reused by `rtt` and `ett`.

## Local Kimai service

[`docker-compose.yml`](docker-compose.yml) is a disposable local Kimai 2.x
service based on the [official Docker Compose guide](https://www.kimai.org/documentation/docker-compose.html): MySQL 8.3, `kimai/kimai2:stable`, a
health-checked database dependency, and Kimai on `http://127.0.0.1:8001`.

It contains **test-only defaults** so a fresh checkout can be started. Override
all `KIMAI_*` values (especially database passwords, admin password, and app
secret) for any shared or non-disposable environment. Do not expose the default
port publicly.

### Start and check readiness

Docker Compose is supported:

```sh
docker compose config
docker compose up -d --wait
curl --fail --location http://127.0.0.1:8001/ >/dev/null
```

Podman Compose is preferred and is supported with the same file:

```sh
podman compose config
podman compose up -d --wait
# Wait until this succeeds: Kimai can still be starting after `up --wait`.
until curl --fail --silent http://127.0.0.1:8001/ >/dev/null; do sleep 2; done
```

On hosts where `podman compose` delegates to an external Compose provider, that
is still the supported Podman entry point; verify with `podman ps` that the
containers are Podman containers. The compose file has no Docker-engine-only
features.

Stop services while retaining test data:

```sh
podman compose down
```

Reset all local test data and volumes:

```sh
podman compose down --volumes --remove-orphans
```

### One-time test fixture

1. Sign in at `http://127.0.0.1:8001/` with the admin credentials supplied to
   Compose (complete the first-run wizard).
2. Create an API access token in **personal menu → API Access**. Copy it when
   Kimai displays it; clients need a Bearer token and the value is not a test
   artifact.
3. Create one visible customer, project, and activity. Record their numeric
   project and activity IDs. A Kimai timesheet requires both IDs.
4. If tags are in scope, create visible tags in Kimai before using them.
5. Configure the client with the base URL (not necessarily `/api`) and the
   token. A client must discover the authenticated user and the single Kimai
   instance scope before normal work begins.

Suggested deterministic names are `E2E Customer`, `E2E Project`, and `E2E
Activity`. Prefix all generated entry descriptions with the client name and
`E2E` so cleanup is safe.

## Provider acceptance scenarios

Use a fresh, dedicated Kimai account/fixture. Record the Kimai version, client
revision, compose command, and the generated entry IDs with the evidence.

| ID | Scenario | Expected provider-visible result |
| --- | --- | --- |
| K01 | Authenticate and discover | Valid token returns the current user and one synthetic instance scope; an invalid token fails without saving a usable session. |
| K02 | Load project/activity choices | The visible fixture project appears; filtering activities by its numeric project ID returns the visible fixture activity. |
| K03 | Empty snapshot | With no active timer, the snapshot says no timer is running and lists recent completed entries only. |
| K04 | Start | A description, project, activity, and billable value create one active timesheet. The active entry has the supplied metadata and a non-empty numeric ID. |
| K05 | Tags (when supported) | Pre-existing tag names are preserved in the created/updated timesheet. Empty tags do not send an invalid tag value. |
| K06 | Running update | Change the description and/or billable value of the active entry. Its ID and begin time remain unchanged; the changed fields persist after refresh. |
| K07 | Stop | Stop the active entry. It has an end time, is no longer returned as active, and appears in recent entries. |
| K08 | Continue | Continue a completed entry. Kimai has a new active entry with copied description, project, activity, billable value, and tags; the source stays completed. |
| K09 | Delete | Delete a known completed test entry. It no longer appears in a refreshed recent-entry list. Never use this scenario against non-fixture data. |
| K10 | Totals and ordering | The active elapsed time advances while running; recent completed entries are newest first; today's total includes the test entries that overlap the local day. |
| K11 | Error recovery | Stop with no active timer, start without project/activity, invalid IDs, network failure, and expired/invalid token produce actionable errors and leave the displayed snapshot consistent after refresh. |

## Interactive UI contract

Run K01–K10 through every interactive frontend, not only its API/CLI layer.
For each frontend verify that the user can:

- see provider name, timer state, elapsed time, today's total, and errors;
- enter a description; select the fixture project and activity; toggle
  billability; start and stop;
- save a running-entry update;
- refresh without losing the actual server state;
- continue and delete a selected recent fixture entry; and
- see a success/failure status after every mutation.

Capture an idle and a running screenshot at minimum, plus a screenshot that
shows successful update/continue/delete feedback when the UI exposes it. Keep
credentials, Authorization headers, and raw Kimai API tokens out of screenshots
and transcripts.

For Fyne/GLFW on this repository, use the direct X11 procedure in
[`docs/VISION-X11.md`](docs/VISION-X11.md): Xvfb + Openbox + `xdotool` and an
FFmpeg `x11grab` screenshot. Do not substitute a Wayland/Weston session for
that test path.

## Evidence and cleanup

Store each run under the implementation's `test-results/<UTC timestamp>/`
directory. Include a short `README.md` containing commands, fixture IDs,
scenario outcomes, environment versions, known limitations, and links to
screenshots/transcripts. Redact secrets before retaining any artifact.

Finish every run by stopping any active fixture timer, deleting only entries
created by the run if a clean instance is desired, and tearing down the local
stack with the chosen Compose command.
