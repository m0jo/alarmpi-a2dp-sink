# alarmpi-a2dp-sink
Raspberry Pi Bluetooth Audio Receiver

This HowTo is meant to show you how to set up your Raspberry Pi to play music shared from your phone, tablet, .. via bluetooth aka A2DP.
Since we usually have no way to display anything on a Raspi, we want to pair it with our audiosource without any PIN/Pass authetication. You may consider this a security risk but given that bluetooth has a typical range of about <10m you may just punch the guy playing techno at 4:00pm at max volume.

We are using Archlinux-ARM here, however this should work on other distributions as well.

### Prerequisites
* Linux compatible USB-Bluetooth device

### Installing required packages
```
$ sudo pacman -S expect alsa-utils alsa-lib pulseaudio pulseaudio-alsa pulseaudio-bluetooth bluez bluez-libs bluez-utils
```
### Configuring audio
```
# gpasswd -a <your-username> audio
# useradd pulse
# gpasswd -a pulse audio
```
Create a new file called pulseaudio.service in /etc/systemd/system/ and paste the following
```
[Unit]
Description=Pulseaudio
Requires=bluetooth.target

[Service]
User=pulse
ExecStart=/usr/bin/pulseaudio -v
Restart=always
LimitRTPRIO=99
LimitNICE=40
LimitMEMLOCK=40000

[Install]
WantedBy=multi-user.target
```
... and enable it at boot
```
# systemctl enable pulseaudio
# systemctl start pulseaudio
```

### Configuring bluetooth

```
# gpasswd -a <your-username> lp
# gpasswd -a pulse lp
# systemctl enable bluetooth
# systemctl start bluetooth
# hciconfig hci0 up
# hciconfig hci0 class 0x200420
```
Edit `/etc/bluetooth/main.conf` and set Class to `0x200420`, this makes your bt-device act like a car audio system.

To power on the bluetooth adapter at boot, create a new file called `10-bluetooth.rules` in `/etc/udev/rules.d/`
and paste the following
```
ACTION=="add", KERNEL=="hci0", RUN+="/usr/bin/hciconfig hci0 up"
```

### Making bluetooth pair without PIN/pass
Create a new file in your $HOME directory called `bt_pair_auto` and paste the following
```
#!/usr/bin/expect -f
spawn "bluetoothctl"
expect "# "
send "discoverable on\r"
expect "Changing discoverable on succeeded"
send "pairable on\r"
expect "Changing pairable on succeeded"
send "agent NoInputNoOutput\r"
expect "Agent registered"
send "default-agent\r"
expect "Default agent request successful"
set timeout -1
while { 1 == 1 } {
expect "Authorize service"
send "yes\r"
}
```
This pairs your Pi with any device that sends a pairing request and allows audio forwarding. Be careful since this script may hide errors from you.

Again, create a new systemd-service to enable the script at boot by creating a new file called `bt_auto_pair.service` in `/etc/systemd/system`

```
[Unit]
Description=bt auto pair with every connecting device
Requires=bluetooth.target

[Service]
User=root
ExecStart=/home/<your-username>/bt_pair_auto

[Install]
WantedBy=multi-user.target
```

and enable it:
```
# systemctl enable bt_auto_pair
# systemctl start bt_auto_pair
```

Make sure to replace all occurrences of `<your-username>` with your actual username.

`reboot` and you should be ready to stream audio from your phone to your raspi.
