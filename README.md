# OLED-Stats

## OLED Stats Display Script For Raspberry Pi

Full setup instructions available on my blog - https://www.the-diy-life.com/add-an-oled-stats-display-to-raspberry-pi-os-bullseye/
Or my Youtube Channel - https://youtu.be/lRTQ0NsXMuw

The script is pre-configured for 128x64 I2C OLED Display, but can easily be modified to run on a 128x32 I2C OLED Display

## I. Screenshots:

<table align="center" style="margin: 0px auto;">
  <tr>
    <th>stats.py</th>
    <th>monitor.py</th>
  </tr>
  <tr>
    <td><img align="right" src="https://i.ytimg.com/vi/lRTQ0NsXMuw/hq720.jpg?sqp=-oaymwEcCOgCEMoBSFXyq4qpAw4IARUAAIhCGAFwAcABBg==&rs=AOn4CLA2eFunUPnMf_Cveih2-b_JEXZxig" height="220"></img></td>
    <td><img align="right" src="https://i.ytimg.com/vi/94ZjxjmhBrY/hq720.jpg?sqp=-oaymwEcCOgCEMoBSFXyq4qpAw4IARUAAIhCGAFwAcABBg==&rs=AOn4CLBTY9ptxf2VqzErucUVVxqmK3Pw6g" height="220"></img></td>
  </tr>
  </table>

## II. Installation Steps:

### 2.1. Connect **GND, VCC(3.3v), SCL, & SDA** ports of the display according to the picture shown below:

<img src="https://www.the-diy-life.com/wp-content/uploads/2021/11/Screenshot-2021-11-14-at-22.16.39-1024x576.jpg">

### 2.2. Upgrade your Raspberry Pi firmware and reboot:

```bash
sudo apt-get update
sudo apt-get full-upgrade
sudo reboot
```

### 2.3. Install python3-pip & upgrade the setuptools

```bash
sudo apt-get install python3-pip
sudo pip3 install --upgrade setuptools
```

### 2.4. Next, we’re going to install the Adafruit CircuitPython library using the following commands:

```bash
cd ~
sudo pip3 install --upgrade adafruit-python-shell
sudo reboot

wget https://raw.githubusercontent.com/adafruit/Raspberry-Pi-Installer-Scripts/master/raspi-blinka.py
sudo python3 raspi-blinka.py
```

### 2.5. Check the `I2C` status using the command:

```bash
sudo i2cdetect -y 1

    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- 3c -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

### 2.6. Next, we need to install the CircuitPython libraries specific to the display. Start by entering the following commands:

```bash
sudo pip3 install adafruit-circuitpython-ssd1306
sudo pip3 install psutil
sudo reboot
sudo apt-get install python3-pil
```

### 2.7. Now we need to download the python script from out github:

```bash
git clone https://github.com/mklements/OLED_Stats.git

cd OLED_Stats
cp PixelOperator.ttf ~/PixelOperator.ttf
cp stats.py ~/stats.py
    
cp psutilstats.py ~/psutilstats.py
    
cp lineawesome-webfont.ttf ~/lineawesome-webfont.ttf
cp monitor.py ~/monitor.py
```

### 2.8. For activating the `crontab` follow the procedure:

```bash
crontab -e
```

**Add this at the bottom:**

Remember to change your username (pi below) if you're not using the default username

```bash
@reboot python3 /home/pi/stats.py &
```
OR

```bash
@reboot python3 /home/pi/psutilstats.py &
```
OR

```bash
@reboot python3 /home/pi/monitor.py &
```

9. At the end DELETE the OLED_Stats folder and reboot

```bash
sudo rm -rf OLED_Stats
sudo reboot
```

### 2.9. Display Issues:

If your display shows jumbled pixels/symbols instead of actual text - you may have a display which supports the SH1106 driver instead of more common SSD1306 driver. This script ONLY works for SSD1306 displays.
If you have this issue, follow this guide instead: https://www.youtube.com/watch?v=LdOKXUDw2NY

<h3><p align="center">THE  END</p></h3>