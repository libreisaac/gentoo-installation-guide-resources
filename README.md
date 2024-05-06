# Gentoo Installation Guide Resources

This repository contains transcripts of my Gentoo installation guide located at https://youtu.be/t0nPxDlFL2I.

## Related Repositories

- Portage Configuration File Templates: https://github.com/libreisaac/portage-config
- Sway Configuration Files: https://github.com/libreisaac/sway-config

## Corrections

### Pipewire User Group
After setting up the Sway desktop environment, you'll want to run `usermod [username] -aG pipewire` to give your user account control over Pipewire.

### WiFi Setup During Installation
Unfortunately I missed a couple steps when describing connecting to WiFi; prior to entering the `wpa_cli`:
- Run `nano /etc/wpa_supplicant/wpa_supplicant.conf` to create a WPA Supplicant configuration file.
- Add `ctrl_interface=/run/wpa_supplicant` on the first line of the file.
-  Add `update_config=1` to the second line of the file.
- Save the file and exit Nano with `CTRL + S` and `CTRL + X`.
- Start the `wpa_supplicant` service with `rc-service wpa_supplicant start`.
Then run `wpa_cli` as described, configuring and enabling a network. Once you've finished, you can run `rc-service wpa_supplicant restart` to connect.
