# **samba-installer**

This example:
- has two phases during installation:
    - init phase, and
    - setup phase.
- sets up `mount` module during installation's init phase.
- installs `samba` package during installation.
- removes samba when uninstalled.

From inside this folder, run:

```
# Install.
./dispatch install

# Setup samba for guest.
./dispatch samba/init-guest-samba

# Uninstall.
./dispatch uninstall
```
