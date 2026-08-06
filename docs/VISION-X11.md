# Direct X11 GUI Inspection and Automation

For this specific Fyne/GLFW binary, direct X11 is substantially simpler and more reliable than running it through Weston/Xwayland. This is the recommended default for coding-agent GUI inspection, screenshots, input automation, and optional host viewing.

Use the Weston procedure in [`VISION.md`](./VISION.md) only when testing actual Wayland/compositor behavior is a product requirement.

## Tested result

This architecture was tested successfully on Fedora 44 in the Lima guest:

```text
Fyne/GLFW application ─┐
Openbox window manager ├─> Xvfb :99 ─┬─> FFmpeg x11grab
xdotool automation ────┘              └─> x11vnc ─> SSH tunnel ─> host viewer
```

Verified behavior:

- Xvfb initialized at `1280x720x24`.
- Openbox placed, focused, and decorated the application normally.
- `gtt-fyne` mapped a visible window named `go-time-tracker`.
- The window rendered correctly rather than as a black surface.
- Mesa provided direct software rendering through llvmpipe without extra flags.
- FFmpeg captured the live desktop directly from `:99.0`.
- `xdotool` focused the Fyne entry and typed `Example description` into it.
- `vncdotool` captured the x11vnc framebuffer successfully.
- Two VNC clients connected concurrently; the second saw `1 other clients` and did not disconnect the first.
- No Weston PAM rule, VNC certificate prompt, nested viewer, or compositor screenshot protocol was needed.

The test produced [`gtt-fyne-x11-screenshot.png`](./gtt-fyne-x11-screenshot.png):

![Direct X11 screenshot of gtt-fyne](./gtt-fyne-x11-screenshot.png)

The screenshot also shows the application-level configuration error expected in this checkout:

```text
Error: gtt is not configured; run `gtt configure`
```

That error is unrelated to X11 rendering.

## Packages

Install the direct-X11 stack with:

```sh
sudo dnf install -y \
  xorg-x11-server-Xvfb \
  openbox \
  x11vnc \
  ffmpeg-free \
  xdotool \
  xdpyinfo \
  xprop \
  glx-utils
```

Mesa software rendering was already present in the tested guest. `glxinfo -B` identified:

```text
direct rendering: Yes
OpenGL vendor string: Mesa
OpenGL renderer string: llvmpipe (LLVM 22.1.8, 256 bits)
OpenGL version string: 4.6 (Compatibility Profile) Mesa 26.1.4
```

## Recommended startup sequence

Run all X11 clients with the same explicit `DISPLAY` value:

```sh
export DISPLAY=:99
```

### 1. Start Xvfb

```sh
Xvfb :99 \
  -screen 0 1280x720x24 \
  -nolisten tcp \
  >xvfb-x11.log 2>&1 &
xvfb_pid=$!
```

Wait for the display instead of relying on a fixed sleep:

```sh
for _ in $(seq 1 100); do
  xdpyinfo >/dev/null 2>&1 && break
  sleep 0.1
done
xdpyinfo >/dev/null
```

`-nolisten tcp` prevents direct network access to the X server. x11vnc is the only remote-display endpoint.

### 2. Start Openbox

```sh
openbox >openbox-x11.log 2>&1 &
wm_pid=$!
```

A window manager is not strictly required to create an X window, but it makes focus, activation, placement, sizing, and decorations predictable. It also makes `xdotool windowactivate --sync` useful.

At `1280x720`, Openbox constrained the application’s requested `1100x720` client size to `1100x693` so that its title bar and borders fit on screen. Nothing important was clipped. Use a taller Xvfb screen if an exact `1100x720` client area is required.

### 3. Start x11vnc

```sh
x11vnc \
  -display :99 \
  -localhost \
  -forever \
  -shared \
  -nopw \
  -rfbport 5900 \
  >x11vnc-x11.log 2>&1 &
vnc_pid=$!
```

Relevant options:

- `-localhost` binds the server for local/tunneled access only.
- `-forever` keeps it running after a viewer disconnects.
- `-shared` permits simultaneous host viewers and automation clients.
- `-nopw` is acceptable only because the endpoint is loopback-only and reached through SSH.

Verify the listener:

```sh
ss -ltnp | grep '127.0.0.1:5900'
```

### 4. Start the application

```sh
DISPLAY=:99 ./gtt/bin/gtt-fyne >gtt-fyne-x11.log 2>&1 &
app_pid=$!
```

Wait for the actual window title:

```sh
window=$(
  DISPLAY=:99 xdotool search \
    --sync \
    --onlyvisible \
    --name '^go-time-tracker$' \
  | head -1
)
```

The tested window properties were:

```text
WM_CLASS = "go-time-tracker", "go-time-tracker"
WM_NAME  = "go-time-tracker"
```

Searching for only `gtt` is not reliable because `gtt` is not a contiguous substring of the window title.

## Host viewing through Lima SSH

Run the tunnel on the host:

```sh
ssh \
  -F /home/user/.lima/default/ssh.config \
  -N -T \
  -L 127.0.0.1:5901:127.0.0.1:5900 \
  lima-default
```

Connect the host VNC viewer to:

```text
127.0.0.1:5901
```

No username or password is required by this temporary x11vnc configuration. The SSH tunnel supplies transport security and access control.

Because x11vnc runs with `-shared`, a guest-side capture or automation client can connect without ejecting the host viewer. This differs from Weston’s VNC backend, which allowed only one viewer at a time.

## Screenshots inside the guest

Capture the Xvfb framebuffer directly:

```sh
DISPLAY=:99 ffmpeg \
  -hide_banner \
  -loglevel error \
  -y \
  -f x11grab \
  -video_size 1280x720 \
  -i :99.0 \
  -frames:v 1 \
  gtt-fyne-x11-screenshot.png
```

This operation reads the same framebuffer used by the application and viewer. It neither opens a new VNC connection nor disturbs existing viewers.

### Application client area only (no title bar, borders, or desktop)

Use the top-level Fyne client window’s geometry as the X11 capture rectangle. This was tested successfully and produced [`gtt-fyne-x11-window.png`](./gtt-fyne-x11-window.png), a `1100x693` PNG containing only application content:

![Client-area-only X11 screenshot of gtt-fyne](./gtt-fyne-x11-window.png)

```sh
window=$(DISPLAY=:99 xdotool search \
  --sync \
  --onlyvisible \
  --name '^go-time-tracker$' \
  | head -1)

eval "$(DISPLAY=:99 xdotool getwindowgeometry --shell "$window")"

DISPLAY=:99 ffmpeg \
  -hide_banner \
  -loglevel error \
  -y \
  -f x11grab \
  -draw_mouse 0 \
  -video_size "${WIDTH}x${HEIGHT}" \
  -i ":99.0+${X},${Y}" \
  -frames:v 1 \
  gtt-fyne-x11-window.png
```

`xdotool getwindowgeometry --shell` supplies the client window’s `X`, `Y`, `WIDTH`, and `HEIGHT`. With Openbox, the test found:

```text
X=1
Y=22
WIDTH=1100
HEIGHT=693
```

The associated frame extents were `left=1`, `right=1`, `top=22`, and `bottom=5`; starting the capture at the client `X,Y` correctly omitted all of those decorations. Do not calculate these values manually: query the current window before each capture because position and size can change.

`-draw_mouse 0` is optional; it removes the mouse pointer for a clean evidence image. Omit it when the pointer location is useful context.

This approach captures a rectangular client area. It does not account for overlapping windows, menus, or dialogs: those X pixels are captured if present in the rectangle. Activate the target window and dismiss transient UI first when a clean application image is required.

x11vnc is also compatible with `vncdotool` in this architecture:

```sh
uvx vncdotool \
  --server 127.0.0.1::5900 \
  --timeout 15 \
  capture /tmp/gtt-fyne-vnc.png
```

Unlike the Weston VNC backend, this returned a valid `1280x720` PNG immediately.

## Input automation

Activate the real application window before sending input:

```sh
window=$(
  DISPLAY=:99 xdotool search \
    --sync \
    --onlyvisible \
    --name '^go-time-tracker$' \
  | head -1
)

DISPLAY=:99 xdotool windowactivate --sync "$window"
DISPLAY=:99 xdotool mousemove 300 160 click 1
DISPLAY=:99 xdotool key ctrl+a
DISPLAY=:99 xdotool type --delay 20 'Example description'
```

The coordinates above target the description field with the tested Openbox placement and `1280x720` screen. Re-read window geometry or use image-based targeting if layout changes.

Fyne draws most widgets into one canvas window. The description field is not a separate native X child window, so tools such as `xdotool search` can find and focus the top-level window but cannot discover individual Fyne controls semantically. Use one of:

- stable coordinates relative to the application window;
- keyboard traversal with Tab/Shift+Tab;
- screenshot/image matching;
- application-level test hooks when available.

## Diagnostics

Check X availability:

```sh
DISPLAY=:99 xdpyinfo >/dev/null
```

Check OpenGL initialization and renderer:

```sh
DISPLAY=:99 glxinfo -B
```

Check that the application mapped a visible window:

```sh
DISPLAY=:99 xdotool search \
  --onlyvisible \
  --name '^go-time-tracker$'
```

Inspect the title and class:

```sh
DISPLAY=:99 xprop -id "$window" WM_CLASS WM_NAME _NET_WM_NAME
```

If GLFW/OpenGL does not initialize with the default environment, force software rendering:

```sh
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  ./gtt/bin/gtt-fyne >gtt-fyne-x11.log 2>&1 &
```

This was not necessary in the tested Fedora guest because Mesa selected llvmpipe automatically.

If a screenshot is black:

1. confirm `xdpyinfo` can connect to `:99`;
2. confirm the app window is visible with `xdotool`;
3. inspect `gtt-fyne-x11.log` and `glxinfo -B`;
4. confirm FFmpeg is capturing `:99.0`, not an SSH-forwarded display;
5. use a 24-bit Xvfb screen and try `LIBGL_ALWAYS_SOFTWARE=1`.

## Cleanup

Retain each background PID and terminate in app-to-server order:

```sh
kill "$app_pid" "$vnc_pid" "$wm_pid" "$xvfb_pid"
```

Wait briefly and use `kill -KILL` only for processes that remain alive. Do not remove `/tmp/.X99-lock` or `/tmp/.X11-unix/X99` while Xvfb is still running. Stale files may be removed only after confirming that no `:99` X server exists.

No PAM or system authentication file is created by this workflow.

## Comparison with the Weston workflow

| Capability | Direct X11/Xvfb | Weston VNC/Xwayland |
| --- | --- | --- |
| Fyne/GLFW compatibility | Native path used by this binary | Requires Xwayland |
| Guest screenshot | Direct `x11grab` | Needed a nested VNC viewer |
| Host and agent connected together | Yes, with `x11vnc -shared` | No, one VNC client at a time |
| `xdotool` automation | Direct on the live framebuffer | Indirect through Xwayland/VNC |
| `vncdotool` capture | Worked | Authenticated, then timed out |
| PAM override | None | Required for the test setup |
| First-use certificate dialog | None | Present in TigerVNC/Weston test |
| Compositor-specific behavior | Not tested | Yes |

The conclusions in [`VISION.md`](./VISION.md) remain accurate when narrowly read as follows:

- capturing the **Weston-composited VNC output** required capturing a VNC viewer;
- Xwayland’s root buffer was not Weston’s composited output;
- Weston’s built-in screenshot options were unsuitable in the tested environment.

They should not be interpreted as evidence that direct X11/Xvfb is unsuitable. Direct X11 was not part of that initial workflow. It is now tested and is the better default for this Fyne binary.

## Recommendation

Use **Xvfb + Openbox + x11vnc** for routine coding-agent work involving this GUI:

1. start Xvfb;
2. start Openbox;
3. start loopback-only, shared x11vnc;
4. start the application on the Xvfb display;
5. inspect or automate with FFmpeg and `xdotool`;
6. optionally tunnel x11vnc to a host viewer;
7. stop all four processes after the task.

Use Weston only for tests where Wayland, Xwayland integration, compositor policy, or Weston-specific rendering is itself under test.
