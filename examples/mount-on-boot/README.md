# **samba-installer**

This example:
- installs `autostart` service during installation.
- sets up `mount` module during installation.
- removes `autostart` service when uninstalled.

`autostart` service will be called each time on boot. Subsequently, `mount` module will be dispatched on boot due to it listening to `autostart` event.

From inside this folder, run:

```
# Install.
./dispatch install

# Setup samba for guest.
source ./query mount

# Check for repo content.
ls $repo_path

# Uninstall.
./dispatch uninstall
```
