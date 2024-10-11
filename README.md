This has been tested on 2016 ThinkPad X270.\
Made for the `ID 138a:0097 Validity Sensors, Inc.` scanner.

## Demo
*Video demo will be added soon™*

#### Successful read:

![Screenshot from 2024-10-11 20-17-05](https://github.com/user-attachments/assets/318a73e1-c801-4d59-b1c0-176dbd84afac)

#### Failed read (and successful in the end):

![Screenshot from 2024-10-11 20-17-17](https://github.com/user-attachments/assets/92f0cdc0-2518-4c6e-97fc-57abd691a350)

# Setup on Arch
#### Firstly, check if you have the right fingerprint sensor by running `lsusb`:
```
~ lsusb
Bus 001 Device 007: ID 138a:0097 Validity Sensors, Inc.
```
(It might work with a different scanner, you can try it.)
#### Install [python-validity](https://github.com/uunicorn/python-validity):
```
yay -S python-validity
```
#### Scan your finger:
```
fprintd-enroll
```
#### Verify it with:
```
fprintd-verify
```
### Unlocking after suspend
Normally, the reader stops working after suspending your laptop, so here is a [fix](https://github.com/uunicorn/python-validity/issues/106#issuecomment-1019342483):
#### Create a service file:
```
sudo nano /etc/systemd/system/restart_fprintd_after_sleep.service
```
#### Paste this code into the file:

```
[Unit]
Description=Restart services to fix fingerprint integration
After=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target

[Service]
Type=oneshot
ExecStart=systemctl restart open-fprintd.service python3-validity.service

[Install]
WantedBy=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```
#### And activate it:
```
sudo systemctl enable --now restart_fprintd_after_sleep.service
```
#### PAM:
Now, to activate the reader after running sudo, add this *on top* of `/etc/pam.d/system-auth`:
```
auth            sufficient      pam_unix.so try_first_pass likeauth nullok
auth            sufficient      pam_fprintd.so
```
You can enable fingerprint login in swaylock, by adding the same code into `/etc/pam.d/swaylock`.\
Or on your login screen ([ly](https://github.com/fairyglade/ly), etc.) (`/etc/pam.d/system-login`), SDDM (`/etc/pam.d/sddm`), ...
# Troubleshooting
<sup>From [python-validity](https://github.com/uunicorn/python-validity?tab=readme-ov-file#error-situations)</sup>

### [List devices failed](https://github.com/uunicorn/python-validity?tab=readme-ov-file#list-devices-failed)
If `fprintd-enroll` returns with 
```
list_devices failed:
```
or 
```
GDBus.Error:net.reactivated.Fprint.error.NoSuchDevice
```
you can check the logs of the `python3-validity` daemon using
```
sudo systemctl status python3-validity
```
If it's not running, you can enable and/or start it by substituting `status` with `enable` or `start`.

### [Errors on startup](https://github.com/uunicorn/python-validity?tab=readme-ov-file#errors-on-startup)
If `systemctl status python3-validity` complains about errors on startup, you may need to factory-reset the fingerprint chip. Do that like so:
```
sudo systemctl stop python3-validity
sudo validity-sensors-firmware
sudo python3 /usr/share/python-validity/playground/factory-reset.py
sudo systemctl start python3-validity
fprintd-enroll
```
At some of the above points you may get a `Device busy` error, depending on how systemctl plays along. Kill offending processes if necessary, or re-run `systemctl stop python3-validity`, in case it has automatically been restarted.