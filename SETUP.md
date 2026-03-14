# Howdy on Zorin OS 18 — Fixes & Custom Additions

Standard Howdy installation docs: https://github.com/boltgolt/howdy

This documents the issues encountered and fixes applied when setting up Howdy on Zorin OS 18 (Ubuntu Noble, X11) with an MSI Cyborg 17 laptop, plus custom additions built on top.

## Fixes Applied

### 1. PPA not available for Zorin OS

Zorin uses a custom codename that the Howdy PPA doesn't recognize. Built from source instead:

```bash
git clone https://github.com/boltgolt/howdy.git ~/howdy
cd ~/howdy && meson setup build && meson compile -C build && sudo meson install -C build
```

### 2. Meson build fails — `/usr/bin/python` not found

Ubuntu Noble only ships `python3`, but Howdy's meson build expects `/usr/bin/python`.

**Fix:**
```bash
sudo ln -sf /usr/bin/python3 /usr/bin/python
```

### 3. PAM module not found after install

Howdy installs `pam_howdy.so` to `/usr/local/lib/x86_64-linux-gnu/security/` but PAM only looks in `/lib/x86_64-linux-gnu/security/`. Face auth silently does nothing.

**Fix:**
```bash
sudo ln -s /usr/local/lib/x86_64-linux-gnu/security/pam_howdy.so \
  /lib/x86_64-linux-gnu/security/pam_howdy.so
```

### 4. `meson install` overwrites config

Every `sudo meson install -C build` replaces `/usr/local/etc/howdy/config.ini` with defaults, resetting `device_path` back to `none`.

**Fix:** Changed the default in the source file `~/howdy/howdy/src/config.ini` so reinstalls keep the correct value:
```ini
device_path = /dev/video0
```

### 5. Howdy GTK video tab shows black screen

Two issues in `howdy-gtk/src/tab_video.py`:

1. OpenCV defaults to the GStreamer backend, which fails silently and produces black frames on this system.
2. The frame rendering used PNG encoding via `PixbufLoader`, which didn't display properly.

**Fix** (in `~/howdy/howdy-gtk/src/tab_video.py`):

Force V4L2 backend:
```python
# Before (broken):
self.capture = cv2.VideoCapture(path)

# After (working):
self.capture = cv2.VideoCapture(path, cv2.CAP_V4L2)
```

Replace PNG encoding with direct pixbuf rendering and BGR→RGB conversion:
```python
# Before (broken):
frame = self.cv2.resize(frame, ...)
retval, buffer = self.cv2.imencode(".png", frame)
loader = pixbuf.PixbufLoader()
loader.write(buffer)
loader.close()
buffer = loader.get_pixbuf()
self.opencvimage.set_from_pixbuf(buffer)

# After (working):
frame = self.cv2.cvtColor(frame, self.cv2.COLOR_BGR2RGB)
frame = self.cv2.resize(frame, ...)
h, w, ch = frame.shape
buf = pixbuf.Pixbuf.new_from_data(
    frame.tobytes(), pixbuf.Colorspace.RGB, False, 8, w, h, w * ch
)
self.opencvimage.set_from_pixbuf(buf)
```

Also added graceful handling when the camera isn't ready:
```python
if not ret or frame is None:
    gobject.timeout_add(50, self.capture_frame)
    return
```

## Custom Additions

### Model name description in GTK UI

The "Add Model" dialog in howdy-gtk gives no explanation of what "model name" means. Added a secondary text to `~/howdy/howdy-gtk/src/tab_models.py`:

```python
dialog.format_secondary_text(_(
    "This is a label to help you identify each face scan "
    "(e.g. \"daytime\", \"evening\", \"glasses on\"). "
    "You can leave it blank for a default name."
))
```

### System Tray App

Built a system tray app since Howdy doesn't include one. Uses AyatanaAppIndicator3 for Zorin/Ubuntu compatibility.

**Location:** `/usr/local/lib/x86_64-linux-gnu/howdy/tray.py`

**Features:**
- Enroll new face (opens terminal)
- List enrolled models
- Test camera
- Open Howdy GTK manager
- Edit config
- Disable/enable Howdy

**Dependency:**
```bash
sudo apt install gir1.2-ayatanaappindicator3-0.1
```

**Autostart:** `~/.config/autostart/howdy-tray.desktop` — launches on login automatically.

## PAM Safety

A backup of the original `/etc/pam.d/common-auth` exists at `/etc/pam.d/common-auth.bak`.

**Before editing PAM:** Always open a root shell (`sudo -i`) in a separate terminal first. If PAM breaks, that shell stays authenticated and you can revert:

```bash
cp /etc/pam.d/common-auth.bak /etc/pam.d/common-auth
```
