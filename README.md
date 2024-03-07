# DWIN_T5UIC1_LCD_E3S1 -> Update for techincal debt

## Python class for the Ender 3 V2 and Ender 3 S1 LCD runing klipper3d with Moonraker 

https://www.klipper3d.org

https://octoprint.org/

https://github.com/arksine/moonraker


## Setup:
   Assuming you used [Mainsail](https://docs.mainsail.xyz/setup/getting-started) or [KIAUH](https://github.com/dw-0/kiauh), most of this is already set up.  

### [Disable Linux serial console](https://www.raspberrypi.org/documentation/configuration/uart.md)
  It's possible the primary UART is assigned to the Linux console. If you wish to use the primary UART for other purposes, you must reconfigure Raspberry Pi OS. This can be done by using raspi-config:

  * Start raspi-config: `sudo raspi-config.`
  * Select option 3 - Interface Options.
  * Select option P6 - Serial Port.
  * At the prompt Would you like a login shell to be accessible over serial? answer 'No'
  * At the prompt Would you like the serial port hardware to be enabled? answer 'Yes'
  * Exit raspi-config and reboot the Pi for changes to take effect.
  
  For full instructions on how to use Device Tree overlays see [this page](https://www.raspberrypi.org/documentation/configuration/device-tree.md). 
  
  In brief, ensure there is a line in the `/boot/config.txt` file to apply a Device Tree overlay.
    
    dtoverlay=disable-bt

### Check if Klipper's Application Programmer Interface (API) is enabled

Open ~/printer_data/systemd/klipper.env and ensure the KLIPPER_ARGS variable includes an something like '-a ~/printer_data/comms/klippy.sock'.
If not, add it then restart your pi or 'klipper.service'.

Again assuming you used Mainsail or KIAUH, this should be by defualt.

### Wire the display 

<img src ="images/Raspberry_Pi_GPIO.png?raw=true" width="800" height="572">

  * Display <-> Raspberry Pi GPIO BCM
  * Rx  =   GPIO14  (Tx)
  * Tx  =   GPIO15  (Rx)
  * Ent =   GPIO13
  * A   =   GPIO19
  * B   =   GPIO26
  * Vcc =   2   (5v)
  * Gnd =   6   (GND)

<img src ="images/GPIO.png?raw=true" width="325" height="75">

Here's a diagram based on my color selection:

<img src ="images/GPIO.png?raw=true" width="325" height="75">

I tried to take some images to help out with this: You don't have to use the color of wiring that I used:

<img src ="images/wire1.jpg?raw=true" width="492" height="208"> 
<img src ="images/wire2.jpg?raw=true" width="492" height="208">


<img src ="images/wire3.png?raw=true" width="400" height="200">

<img src ="images/wire4.png?raw=true" width="400" height="300">

I have added some Ender 3S1 specific images:

<img src ="images/Ender3S1_LCD_Board.JPG?raw=true" width="325" height="200">
<img src ="images/Ender3S1_LCD_plug.jpg?raw=true" width="325" height="220">


### Library requirements - Install these now

  Thanks to [wolfstlkr](https://www.reddit.com/r/ender3v2/comments/mdtjvk/octoprint_klipper_v2_lcd/gspae7y)

  `sudo apt-get install python3-pip python3-gpiozero python3-serial git`

  `sudo pip3 install multitimer`

  `git clone https://github.com/nojoyy/KLIPPER_E3S1_LCD.git`

## Run the Code

### Configure API

Enter the downloaded KLIPPER_E3S1_LCD folder.

To get  your API key run:

```bash
~/moonraker/scripts/fetch-apikey.sh
```

Edit the file run.py and paste your API key and verify that you have the proper API_Endpoint address.
```bash
nano run.py
```
This is how the run.py looks for Ender 3 S1, please modify 'ender' with your username (i.e. pi, klipper, etc.)

```python
#!/usr/bin/env python3
from dwinlcd import DWIN_LCD

encoder_Pins = (26, 19)
button_Pin = 13
LCD_COM_Port = '/dev/ttyAMA0'
API_Key = 'XXXXXX'
API_Endpoint = '/home/ender/printer_data/comms/klippy.sock'

DWINLCD = DWIN_LCD(
	LCD_COM_Port,
	encoder_Pins,
	button_Pin,
	API_Key,
	API_Endpoint
)
```

If your control wheel is reversed (Voxelab Aquila) change the encoder_pins to this instead.

```python
#!/usr/bin/env python3
from dwinlcd import DWIN_LCD

encoder_Pins = (19, 26)
button_Pin = 13
LCD_COM_Port = '/dev/ttyAMA0'
API_Key = 'XXXXXX'
API_Endpoint = '/home/ender/printer_data/comms/klippy.sock'

DWINLCD = DWIN_LCD(
	LCD_COM_Port,
	encoder_Pins,
	button_Pin,
	API_Key,
	API_Endpoint
)
```

### Finding Endpoint

Assuming your endpoint doesn't match the provided default, it can be found via the 'klipper.env' file referenced in the EnvironmentFile listed in '/etc/systemd/system/klipper.service' or the direct ExecStart via the -a parameter.

Example Service:
```
[Unit]
Description=Klipper 3D Printer Firmware SV1
Documentation=https://www.klipper3d.org/
After=network-online.target
Wants=udev.target

[Install]
WantedBy=multi-user.target

[Service]
Type=simple
User=ender
RemainAfterExit=yes
WorkingDirectory=/home/ender/klipper
EnvironmentFile=/home/ender/printer_data/systemd/klipper.env
ExecStart=/home/ender/klippy-env/bin/python $KLIPPER_ARGS
Restart=always
RestartSec=10
```

### Verify Connection
Once modified, ensure run.py is executable

```
sudo chmod +x run.py
```

Run with `python3 ./run.py` or './run.py'
Your output should be:

```
DWIN handshake 
DWIN OK.
http://127.0.0.1:80
Waiting for connect to $API_endpoint

Connection.

Boot looks good
Testing Web-services
Web site exists
```

Press ctrl+c to exit run.py

If the test fails to connect, make be sure to double check the existence of the endpoint files or other files in the klipper.env file.

### Run at boot:

    Note: Delay of 20s after boot to allow webservices to settal.

The run.sh script that is loaded by simpleLCD.service will re-run run.py on firmware restarts of the printe. If it fails to start for 5 times within 30 second it will exit and stop until the net boot.

Note if you cloned the repo anywhere other than /home/ender, you will need to modify the 'simpleLCD.service' and 'run.sh' prior to copying 'simpleLCD.service' to /lib/systemd/service.

Just modify 'simpleLCD.service' to point to the correct path of the script.

You will also need to modify run.sh to point to the directory you installed the files of the repo to.

Afterwards, ensure execute permissions:

```bash
sudo chmod +x run.py run.sh simpleLCD.service
```

Copy to systemd:

```bash
sudo cp simpleLCD.service /lib/systemd/system/simpleLCD.service
```

set system permissions:

```bash
sudo chmod 644 /lib/systemd/system/simpleLCD.service
```

Reload daemon and enable and start simpleLCD service
```bash
sudo systemctl daemon-reload && sudo systemctl enable --now simpleLCD.service
```

If service did not start, check status via
```bash
sudo systemctl status simpleLCD.service
```

Worst comes to worst:
```bash
sudo reboot
```

Your LCD should start after 30 seconds. And when you restart your printer firmware the LCD should restart as well.

# Status:

## Working:

 Print Menu:
 
    * List / Print jobs from OctoPrint / Moonraker
    * Auto swiching from to Print Menu on job start / end.
    * Display Print time, Progress, Temps, and Job name.
    * Pause / Resume / Cancle Job
    * Tune Menu: Print speed & Temps

 Perpare Menu:
 
    * Move / Jog toolhead
    * Disable stepper
    * Auto Home
    * Z offset (PROBE_CALIBRATE)
    * Preheat
    * cooldown
 
 Info Menu
 
    * Shows printer info.

## Notworking:
    * Save / Loding Preheat setting, hardcode on start can be changed in menu but will not retane on restart.
    * The Control: Motion Menu
