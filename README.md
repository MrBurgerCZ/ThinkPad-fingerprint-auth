> [!IMPORTANT]
> This has been tested on 2016 ThinkPad X270.\
> Made for the `ID 138a:0097 Validity Sensors, Inc.` scanner.

## Demo
#### Demo
https://www.youtube.com/watch?v=oYyE5cBVhgk

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
> [!NOTE]
> You might need to [install](https://github.com/3v1n0/libfprint) `libfprint-vfs009x-git` in addition if it doesn't work alone, I am not sure yet
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
You can look at this ["issue"](https://github.com/MrBurgerCZ/ThinkPad-fingerprint-auth/issues/1#issue-3265855179), it should help you. Thanks [bc09880](https://github.com/bc09880)! :)

Here is the most important part:
> #### 1. Clear the Fingerprint Database
> 
> The first thing I tried was deleting all registered fingerprints, as this is a common step for resolving issues with corrupted records. This is necessary after successfully enabling the `python3-validity` service:
> 
> fprintd-delete $USER
> 
> **Expected result**: `Fingerprints deleted on DBus driver`
> #### 2. Verify that No Fingerprints Were Registered
> 
> After clearing the database, I checked that no fingerprints remained registered by running:
> 
> fprintd-verify
> 
> **Expected result**: `No fingers enrolled for this device.`
> #### 3. Try Registering the Fingerprints Again
> 
> After clearing the database and confirming it was empty, I resumed the fingerprint registration process. I first registered the left index finger (which worked without issues), then I tried registering the right index finger.
> ##### Register Left Index Finger:
> 
> fprintd-enroll -f left-index-finger
> 
> **Expected result**: `enroll-completed` (successful registration)
> ##### Register Right Index Finger (after clearing the database):
> 
> fprintd-enroll -f right-index-finger
> 
> **Expected result**: `enroll-completed` (successful registration after clearing the database)
> ### Lessons Learned
> #### What DIDN'T Work:
> 
>     * **Simply running `fprintd-enroll`** did not resolve the issue. The fingerprint registration wouldn’t complete, and the error kept occurring.
> 
>     * Trying to register **right-index-finger** multiple times without clearing the database.
> 
>     * Using the incorrect command `sudo fprintd-delete right-index-finger` (instead of clearing the entire database with `$USER`).
> 
>     * Restarting the services alone without clearing the database.
> 
> 
> #### What DID Work:
> 
>     * **Completely clearing** the database with `fprintd-delete $USER`.
> 
>     * **Starting fresh** with all records deleted.
> 
>     * First registering the finger that worked (left index), and then the problematic one (right index).
> 
# Troubleshooting 2
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
