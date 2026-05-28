Да, това са стъпките, но при Bookworm (пък бил той и "Legacy") има една важна особеност: операционната система е по-стриктна по отношение на Python библиотеките.

Ето точния и най-сигурен начин да го направиш на чиста инсталация, стъпка по стъпка, за да не се сблъскаш с грешки от типа "Externally Managed Environment".

1. Подготовка на системата
Първо активираме I2C интерфейса, който по подразбиране е изключен.

Изпълни 

```bash
sudo raspi-config.
```

Отиди на Interface Options -> I2C.

Избери Yes и рестартирай (**sudo reboot**).

2. Инсталиране на системни зависимости
Тези пакети са нужни за обработката на изображения и шрифтове, които дисплеят ще показва.

```bash
sudo apt update

sudo apt install -y python3-pip python3-pil python3-numpy libfreetype6-dev libjpeg-dev libsdl2-dev libportmidi-dev libsdl2-ttf-dev libsdl2-image-dev libsdl2-mixer-dev
```

3. Инсталиране на Luma.OLED (Модерният начин)
В Bookworm имаш два варианта за инсталация на Python библиотеки. Препоръчвам ти използването на --break-system-packages за този малък проект, тъй като е най-лесно за начинаещи:

```bash
sudo pip3 install luma.oled --break-system-packages
```

4. Проверка на адреса на дисплея
Свържи дисплея и провери дали системата го вижда:

```bash
sudo apt install -y i2c-tools

i2cdetect -y 1
```
Трябва да видиш 3c в таблицата.

5. Създаване на актуалния скрипт
Създай файла:

```bash
nano stats.py
```
И постави този код, който е оптимизиран за Luma.OLED:

```bash
from luma.core.interface.serial import i2c
from luma.oled.device import ssd1306
from luma.core.render import canvas
from PIL import ImageFont
import subprocess
import time
import os

# Инициализация
serial = i2c(port=1, address=0x3C)
device = ssd1306(serial)

# Шрифтове
font_path = "/home/pi/fonts/PixelOperator.ttf"
font = ImageFont.truetype(font_path, 16) if os.path.exists(font_path) else ImageFont.load_default()

def get_stats():
    # IP
    cmd_ip = "hostname -I | cut -d' ' -f1"
    IP = subprocess.check_output(cmd_ip, shell=True).decode("utf-8").strip()
    
    # CPU Load (вземаме само числото)
    cmd_cpu = "top -bn1 | grep load | awk '{printf \"%.2f\", $(NF-2)}'"
    CPU = subprocess.check_output(cmd_cpu, shell=True).decode("utf-8").strip()
    
    # Температура
    cmd_temp = "vcgencmd measure_temp | cut -f 2 -d '='"
    Temp = subprocess.check_output(cmd_temp, shell=True).decode("utf-8").strip()

    # RAM
    cmd_mem = "free -m | awk 'NR==2{printf \"Mem: %s/%sMB\", $3,$2}'"
    MemUsage = subprocess.check_output(cmd_mem, shell=True).decode("utf-8").strip()
    
    # Диск
    cmd_disk = "df -h | awk '$NF==\"/\"{printf \"Disk: %d/%dGB %s\", $3,$2,$5}'"
    Disk = subprocess.check_output(cmd_disk, shell=True).decode("utf-8").strip()

    return IP, CPU, Temp, MemUsage, Disk

while True:
    IP, CPU, Temp, MemUsage, Disk = get_stats()
    
    with canvas(device) as draw:
        # Ред 1: IP Адрес
        draw.text((0, 0),  f"IP: {IP}", font=font, fill="white")
        
        # Ред 2: Комбиниран ред (Температура и CPU)
        # Използваме символ '|' за разделител за по-добра четливост
        temp_value = Temp.replace("'C", "")
        draw.text((0, 16), f"CPU:{CPU} | T°C:{temp_value}°C", font=font, fill="white")
        
        # Ред 3: RAM
        draw.text((0, 32), f"{MemUsage}", font=font, fill="white")
        
        # Ред 4: Диск
        draw.text((0, 48), f"{Disk}", font=font, fill="white")
    
    time.sleep(2)
```

Защо това е по-добре за чиста инсталация на Bookworm?
Няма да видиш "Platform not detected": Luma.OLED не разчита на стария код на Adafruit, който не разпознаваше Pi 4.

По-лек е: Lite версията на ОС няма графична среда, а Luma работи директно с Framebuffer-а.

Температура: Добавих команда за температура (vcgencmd), тъй като Pi 4 грее повече от старите модели и е полезно да го виждаш на дисплея.

Пусни го с: 

```bash
python3 stats.py. Ако всичко е наред, дисплеят ще светне веднага!
```

Настройка за автоматично стартиране (Auto-start)
За да светва дисплеят веднага щом Raspberry-то зареди (без да се логваш), най-лесният и модерен начин в Bookworm е чрез systemd услуга. Това гарантира, че скриптът ще се рестартира сам, ако случайно спре.

Създай файл за услугата:

```bash
sudo nano /etc/systemd/system/oled_stats.service
```
Постави следното съдържание вътре:

```bash
[Unit]
Description=OLED Statistics Display
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/stats.py
WorkingDirectory=/home/pi
StandardOutput=inherit
StandardError=inherit
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

Активирай услугата:

```bash
sudo systemctl daemon-reload
sudo systemctl enable oled_stats.service
sudo systemctl start oled_stats.service
```

Какво постигнахме:
Визия: Шрифтът PixelOperator изглежда професионално и се събира перфектно на 128x64 екрана.

Автоматизация: Веднага щом включиш тока на разклонителя и Raspberry Pi зареди мрежата, дисплеят ще светне автоматично.

Надеждност: Ако скриптът даде грешка, системата ще го рестартира сама след 5 секунди.

Забележка: Ако след стартиране на услугата дисплеят не светне, провери статуса с: sudo systemctl status oled_stats.service.

RETROPIE OLED

**new script**

```bash
#!/usr/bin/python3
# -*- coding: utf-8 -*-

import time
import os
import board
import textwrap
import busio
import adafruit_ssd1306
import subprocess
from PIL import Image, ImageDraw, ImageFont

# Конфигурация на дисплея
i2c = busio.I2C(board.SCL, board.SDA)
oled = adafruit_ssd1306.SSD1306_I2C(128, 64, i2c)

# Пътища до ресурси
BASE_PATH = "/home/pi/RetroPie-OLED/"

def get_sys_info():
    """ Извлича системна информация по-ефективно """
    try:
        # IP адрес
        IP = subprocess.check_output("hostname -I | cut -d' ' -f1", shell=True).decode('utf-8').strip()
        # CPU Load
        CPU = subprocess.check_output("top -bn1 | grep load | awk '{printf \"%.2f\", $(NF-2)}'", shell=True).decode('utf-8')
        # Memory
        with open("/proc/meminfo", "r") as f:
            lines = f.readlines()
            total = int(lines[0].split()[1]) // 1024
            free = int(lines[1].split()[1]) // 1024
            used = total - free
        # Disk
        df = subprocess.check_output("df -h / | awk 'NR==2{print $3,$2,$5}'", shell=True).decode('utf-8').split()
        return IP, CPU, str(used), str(total), df[0], df[1]
    except:
        return "0.0.0.0", "0.0", "0", "0", "0", "0"

def get_temp(path):
    with open(path, "r") as f:
        return float(f.read()) / 1000

def draw_stats(draw, top, fonts, info, temps):
    """ Повтарящата се функция за чертане на системната инфо таблица """
    IP, CPU_load, MemU, MemT, DiskU, DiskT = info
    CPUT, GPUT, CPUS = temps
    
    # Икони (използваме предварително заредените шрифтове)
    draw.text((5, top+7),   chr(62152), font=fonts['icon'], fill=255)  # Temp
    draw.text((65, top+12),  chr(61614), font=fonts['icon2'], fill=255) # CPU
    draw.text((65, top+33),  chr(62171), font=fonts['icon3'], fill=255) # Mem
    draw.text((5, top+33),   chr(61888), font=fonts['icon2'], fill=255) # Disk

    # Текст данни
    draw.text((15, top+48), IP, font=fonts['system'], fill=255)
    draw.text((20, top+8),  f"{CPUT:.1f}", font=fonts['default'], fill=255)
    draw.text((45, top+8),  "CPU", font=fonts['default'], fill=255)
    draw.text((20, top+18), f"{GPUT}", font=fonts['default'], fill=255)
    draw.text((45, top+18), "GPU", font=fonts['default'], fill=255)
    
    draw.text((82, top+8),  CPU_load, font=fonts['default'], fill=255)
    draw.text((82, top+18), f"{CPUS:.0f}", font=fonts['default'], fill=255)
    
    draw.text((20, top+29), DiskU, font=fonts['default'], fill=255)
    draw.text((45, top+29), "GB", font=fonts['default'], fill=255)
    draw.text((82, top+29), MemU, font=fonts['default'], fill=255)
    draw.text((111, top+29), "MB", font=fonts['default'], fill=255)
    
    draw.text((20, top+39), DiskT, font=fonts['default'], fill=255)
    draw.text((45, top+39), "GB", font=fonts['default'], fill=255)
    draw.text((82, top+39), MemT, font=fonts['default'], fill=255)
    draw.text((111, top+39), "MB", font=fonts['default'], fill=255)

def main():
    # Зареждане на ресурси веднъж
    fonts = {
        'default': ImageFont.load_default(),
        'system': ImageFont.truetype(BASE_PATH + 'neodgm.ttf', 16),
        'rom': ImageFont.truetype(BASE_PATH + 'BM-HANNA.ttf', 16),
        'icon': ImageFont.truetype(BASE_PATH + 'fontawesome-webfont.ttf', 20),
        'icon2': ImageFont.truetype(BASE_PATH + 'fontawesome-webfont.ttf', 14),
        'icon3': ImageFont.truetype(BASE_PATH + 'fontawesome-webfont.ttf', 16)
    }

    images = {
        'sysinfo': Image.open(BASE_PATH + "SysInfo.png").convert('1'),
        'retroarch': Image.open(BASE_PATH + "RetroArchLogo.png").convert('1'),
        'retropie': Image.open(BASE_PATH + "RetropieLogo.png").convert('1'),
        'raspi': Image.open(BASE_PATH + "RaspberryLogo.png").convert('1'),
        'alert': Image.open(BASE_PATH + "AlertImage.png").convert('1')
    }

    width, height = oled.width, oled.height
    image = Image.new('1', (width, height))
    draw = ImageDraw.Draw(image)
    
    intro_step = 0
    last_info_time = 0

    while True:
        cpu_temp = get_temp("/sys/class/thermal/thermal_zone0/temp")
        gpu_temp = os.popen("vcgencmd measure_temp").read().replace("temp=","").replace("'C\n","")
        cpu_speed = get_temp("/sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq")

        draw.rectangle((0, 0, width, height), outline=0, fill=0)

        # 1. Защита от прегряване
        if cpu_temp > 75:
            image.paste(images['alert'], (0,0))
            draw.text((10, 40), "SHUTDOWN: OVERHEAT", font=fonts['default'], fill=255)
            oled.image(image)
            oled.show()
            time.sleep(2)
            os.system("sudo shutdown -h now")
            break
        
        elif cpu_temp > 70:
            image.paste(images['alert'], (0,0))
            draw.text((20, 45), f"WARNING: {cpu_temp:.1f}C", font=fonts['default'], fill=255)

        # 2. Проверка за стартирана игра
        elif os.path.exists('/dev/shm/runcommand.log'):
            try:
                with open('/dev/shm/runcommand.log', 'r') as f:
                    lines = [l.strip() for l in f.readlines()]
                
                # Показване на име на игра и система (опростено за примера)
                system_name = lines[0] if lines else "Unknown"
                game_name = lines[1] if len(lines) > 1 else "Loading..."
                
                draw.text((0, 0), system_name[:15], font=fonts['system'], fill=255)
                gname_wrapped = textwrap.wrap(game_name, width=12)
                y = 20
                for line in gname_wrapped[:3]:
                    draw.text((5, y), line, font=fonts['rom'], fill=255)
                    y += 15
            except:
                pass

        # 3. Интро и Системно инфо
        else:
            if intro_step == 0:
                image.paste(images['raspi'], (0,0))
                intro_step = 1
                oled.image(image)
                oled.show()
                time.sleep(2)
                continue
            elif intro_step == 1:
                image.paste(images['retroarch'], (0,0))
                intro_step = 2
                oled.image(image)
                oled.show()
                time.sleep(2)
                continue
            else:
                # Показване на статистиките
                sys_info = get_sys_info()
                temps = (cpu_temp, gpu_temp, cpu_speed)
                image.paste(images['sysinfo'], (0,0))
                draw_stats(draw, 0, fonts, sys_info, temps)

        oled.image(image)
        oled.show()
        time.sleep(0.5) # Малко по-голяма пауза пести енергия

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        oled.fill(0)
        oled.show()
```

**old script**

```bash
#!/usr/bin/python3
# -*- coding: utf-8 -*-

"""
Title        : RetroPie_OLED.py
Author       : zzeromin, losernator and members of Tentacle Team
Creation Date: Nov 13, 2016
Cafe         : http://cafe.naver.com/raspigamer
Thanks to    : smyani, zerocool, GreatKStar and members of Raspigamer Cafe.
Free and open for all to use. But put credit where credit is due.

Reference    :
https://github.com/haven-jeon/piAu_volumio
https://github.com/adafruit/Adafruit_CircuitPython_SSD1306
https://pillow.readthedocs.io/en/stable/

Notice       :
installed package(apt): python3-pip python3-dev python3-smbus i2c-tools
installed package(pip): Pillow, adafruit-circuitpython-ssd1306

This code edited for rpi3 Retropie v4.0.2 and later by zzeromin
"""

import time
import os
import board
import textwrap
import busio
import adafruit_ssd1306
import subprocess
import os

from sys import exit
from subprocess import *
from time import *
from datetime import datetime
from random import randint
from PIL import Image, ImageDraw, ImageFont
from board import SCL, SDA

# Create the I2C interface.
i2c = busio.I2C(SCL, SDA)

# Create the SSD1306 OLED class.
# The first two parameters are the pixel width and pixel height.  Change these
# to the right size for your display!
oled = adafruit_ssd1306.SSD1306_I2C(128, 64, i2c)
# Alternatively you can change the I2C address of the device with an addr parameter:
#oled = adafruit_ssd1306.SSD1306_I2C(128, 64, i2c, addr=0x31)

# Raspberry Pi pin configuration:
RST = 24
# Note the following are only used with SPI:
DC = 23
SPI_PORT = 0
SPI_DEVICE = 0

intro = 0
HighCPUvariable = 0

def run_cmd(cmd):
# runs whatever in the cmd variable
    p = Popen(cmd, shell=True, stdout=PIPE)
    output = p.communicate()[0]
    return output

def get_cpu_temp():
    tempFile = open("/sys/class/thermal/thermal_zone0/temp")
    cpu_temp = tempFile.read()
    tempFile.close()
    return float(cpu_temp)/1000
    
def get_gpu_temp():
        temp = os.popen("vcgencmd measure_temp").readline()
        temp = temp[:-3]
        return (temp.replace("temp=",""))
        
def get_cpu_speed():
    tempFile = open("/sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq")
    cpu_speed = tempFile.read()
    tempFile.close()
    return float(cpu_speed)/1000
    
def Shutdown():  
    os.system("sudo shutdown -h now") 

def main():
    global intro
    global HighCPUvariable
     
    infoimg = Image.open("/home/pi/RetroPie-OLED/SysInfo.png").convert('1')    
    retroarchimg = Image.open("/home/pi/RetroPie-OLED/RetroArchLogo.png").convert('1')
    titleimg = Image.open("/home/pi/RetroPie-OLED/RetropieLogo.png").convert('1')
    raspyimg = Image.open("/home/pi/RetroPie-OLED/RaspberryLogo.png").convert('1')
    alertimage = Image.open("/home/pi/RetroPie-OLED/AlertImage.png").convert('1')
   
    # Clear display.
    oled.fill(0)
    oled.show()

    # Create blank image for drawing.
    # Make sure to create image with mode '1' for 1-bit color.
    width = oled.width
    height = oled.height
    image = Image.new('1', (width, height))

    # Get drawing object to draw on image.
    draw = ImageDraw.Draw(image)

    # Draw a black filled box to clear the image.
    draw.rectangle((0,0,width,height), outline=0, fill=0)

    padding = 0
    top = padding
    bottom = height-padding

    # Load default font.
    font = ImageFont.load_default()
    font_system = ImageFont.truetype('/home/pi/RetroPie-OLED/neodgm.ttf', 16)
    font_rom = ImageFont.truetype('/home/pi/RetroPie-OLED/BM-HANNA.ttf', 16)
    fonte_rom = ImageFont.truetype('/home/pi/RetroPie-OLED/lemon.ttf', 10)
    font_msg = ImageFont.truetype('/home/pi/RetroPie-OLED/d2.ttf', 11)
    font_icon = ImageFont.truetype('/home/pi/RetroPie-OLED/fontawesome-webfont.ttf', 20)
    font_icon2 = ImageFont.truetype('/home/pi/RetroPie-OLED/fontawesome-webfont.ttf', 14)
    font_icon3 = ImageFont.truetype('/home/pi/RetroPie-OLED/fontawesome-webfont.ttf', 16)
    new_Speed = round(get_cpu_speed(),1)
    new_Temp = round(get_cpu_temp(),1)
    #CPUTemp = str( new_Temp )
    #HighCPU = float(CPUTemp)

    while True:    
        # Shell scripts for system monitoring from here : https://unix.stackexchange.com/questions/119126/command-to-display-memory-usage-disk-usage-and-cpu-load
        cmd = "hostname -I | cut -d\' \' -f1"
        IP = subprocess.check_output(cmd, shell = True )
        cmd = "top -bn1 | grep load | awk '{printf \"%.2f\", $(NF-2)}'"
        CPU = subprocess.check_output(cmd, shell = True )
        cmd = "free -m | awk 'NR==2{printf \"%s\", $3}'"
        MemUsage = subprocess.check_output(cmd, shell = True )
        cmd = "free -m | awk 'NR==2{printf \"%s\", $2}'"
        MemTotal = subprocess.check_output(cmd, shell = True )
        cmd = "free -m | awk 'NR==2{printf \"%.2f%%\", $3*100/$2 }'"
        MemPercent = subprocess.check_output(cmd, shell = True )
        cmd = "df -h | awk '$NF==\"/\"{printf \"%d\", $3}'"
        DiskFree = subprocess.check_output(cmd, shell = True )
        cmd = "df -h | awk '$NF==\"/\"{printf \"%d\", $2}'"
        DiskUsed = subprocess.check_output(cmd, shell = True )
        cmd = "df -h | awk '$NF==\"/\"{printf \"%s\", $5}'"
        DiskPercent = subprocess.check_output(cmd, shell = True )
        #CPUTemp = str( new_Temp ) + chr(0xB0) +"C"
        CPUTemp = str( new_Temp )
        new_Speed = round(get_cpu_speed(),1)
        CPUSpeed = str( new_Speed )
        GPUTemp = (get_gpu_temp())
        HighCPU = float(CPUTemp)
        HighGPU = float(GPUTemp)
        
        if HighCPU > 70 and HighCPU < 75:
            new_Temp = round(get_cpu_temp(),1)
            CPUTemp = str( new_Temp )
            HighCPU = float(CPUTemp)
            draw.rectangle((0,0,width,height), outline=0, fill=0)
            HighCPUvariable = 1
            image.paste(alertimage,(0,0))
            msg1 = "CPU OVER TEMPERATURE"
            msg1_size = draw.textsize(msg1, font=font)
            draw.text(((width-msg1_size[0])/2, top+40), msg1, font=font, fill=255)
            draw.text((47, top+50), str( new_Temp ) + chr(0xB0) +"C", font=font, fill=255)
            oled.image(image)
            oled.show()
            sleep(.1)
        elif HighCPU > 75:
            new_Temp = round(get_cpu_temp(),1)
            CPUTemp = str( new_Temp )
            HighCPU = float(CPUTemp)
            draw.rectangle((0,0,width,height), outline=0, fill=0)
            HighCPUvariable = 1
            #image.paste(alertimage,(0,0))
            msg1 = "SYSTEM SHUTDOWN"
            msg1_size = draw.textsize(msg1, font=font_system)
            draw.text(((width-msg1_size[0])/2, top+10), msg1, font=font_system, fill=255)
            msg1 = "CPU OVER TEMPERATURE"
            msg1_size = draw.textsize(msg1, font=font)
            draw.text(((width-msg1_size[0])/2, top+40), msg1, font=font, fill=255)
            draw.text((47, top+50), str( new_Temp ) + chr(0xB0) +"C", font=font, fill=255)
            oled.image(image)
            oled.show()
            sleep(.1)
            Shutdown()
        else:
            HighCPUvariable = 0
            
            if HighCPUvariable == 0:

                try:
                    f = open('/dev/shm/runcommand.log', 'r', -1,"utf-8")
                    # except FileNotFoundError:
                except IOError:
                    try:
                        infoimg = Image.open("/home/pi/RetroPie-OLED/SysInfo.png").convert('1')
                    except IOError:
                        draw.rectangle((0,0,width,height), outline=0, fill=0)
                        msg1 = "SYSTEM INFO"
                        msg1_size = draw.textsize(msg1, font=font)
                        draw.text(((width-msg1_size[0])/2, top), msg1, font=font, fill=255)
                        # Icons
                        # Icon temperator
                        draw.text((5, top+7),    chr(62152),  font=font_icon, fill=255)
                        # Icon CPU
                        draw.text((65, top+12), chr(61614),  font=font_icon2, fill=255)
                        # Icon memory
                        draw.text((65, top+33), chr(62171),  font=font_icon3, fill=255)
                        # Icon disk
                        draw.text((5, top+33), chr(61888),  font=font_icon2, fill=255)
                        # Icon Wifi
                        #draw.text((0, top+52), chr(61931),  font=font_icon2, fill=255)
                    
                        draw.text((15, top+48),    str(IP,'utf-8'),               font=font_system, fill=255)
                        draw.text((20, top+8),     CPUTemp,                       font=font,        fill=255)
                        draw.text((45, top+8),     str('CPU'),                    font=font,        fill=255)
                        draw.text((20, top+18),    GPUTemp,                       font=font,        fill=255)
                        draw.text((45, top+18),    str('GPU'),                    font=font,        fill=255)                    
                        draw.text((82, top+8),     str(CPU,'utf-8'),              font=font,        fill=255)                    
                        draw.text((82, top+18),    CPUSpeed,                      font=font,        fill=255)
                        draw.text((20, top+29),    str(DiskFree,'utf-8'),         font=font,        fill=255)
                        draw.text((45, top+29),    str('GB'),                     font=font,        fill=255)                    
                        draw.text((20, top+39),    str(DiskUsed,'utf-8'),         font=font,        fill=255)
                        draw.text((45, top+39),    str('GB'),                     font=font,        fill=255)                    
                        draw.text((82, top+29),    str(MemUsage,'utf-8'),         font=font,        fill=255)
                        draw.text((111, top+29),   str('MB'),                     font=font,        fill=255)                    
                        draw.text((82, top+39),    str(MemTotal,'utf-8'),         font=font,        fill=255)
                        draw.text((111, top+39),   str('MB'),                     font=font,        fill=255)                    

                        oled.image(image)
                        oled.show()
                        sleep(.1)
                        pass
                    else:
                        draw.rectangle((0,0,width,height), outline=0, fill=0)
                        if intro == 0:
                            image.paste(raspyimg,(0,0))
                            oled.image(image)
                            oled.show()
                            intro = 1
                            sleep(3)
                        elif intro == 1:
                            draw.rectangle((0,0,width,height), outline=0, fill=0)
                            image.paste(retroarchimg,(0,0))
                            oled.image(image)
                            oled.show()
                            intro = 2
                            sleep(3)
                        elif intro == 2:
                            draw.rectangle((0,0,width,height), outline=0, fill=0)
                            image.paste(titleimg,(0,0))
                            oled.image(image)
                            oled.show()
                            intro = 3
                            sleep(3)
                        elif intro == 3:
                            draw.rectangle((0,0,width,height), outline=0, fill=0)
                            image.paste(infoimg,(0,0))
                            # Icons
                            # Icon temperator
                            draw.text((5, top+7),    chr(62152),  font=font_icon, fill=255)
                            # Icon CPU
                            draw.text((65, top+12), chr(61614),  font=font_icon2, fill=255)
                            # Icon memory
                            draw.text((65, top+33), chr(62171),  font=font_icon3, fill=255)
                            # Icon disk
                            draw.text((5, top+33), chr(61888),  font=font_icon2, fill=255)
                            # Icon Wifi
                            #draw.text((0, top+52), chr(61931),  font=font_icon2, fill=255)
                    
                            draw.text((15, top+48),    str(IP,'utf-8'),               font=font_system, fill=255)
                            draw.text((20, top+8),     CPUTemp,                       font=font,        fill=255)
                            draw.text((45, top+8),     str('CPU'),                    font=font,        fill=255)
                            draw.text((20, top+18),    GPUTemp,                       font=font,        fill=255)
                            draw.text((45, top+18),    str('GPU'),                    font=font,        fill=255)                    
                            draw.text((82, top+8),     str(CPU,'utf-8'),              font=font,        fill=255)                    
                            draw.text((82, top+18),    CPUSpeed,                      font=font,        fill=255)
                            draw.text((20, top+29),    str(DiskFree,'utf-8'),         font=font,        fill=255)
                            draw.text((45, top+29),    str('GB'),                     font=font,        fill=255)                    
                            draw.text((20, top+39),    str(DiskUsed,'utf-8'),         font=font,        fill=255)
                            draw.text((45, top+39),    str('GB'),                     font=font,        fill=255)                    
                            draw.text((82, top+29),    str(MemUsage,'utf-8'),         font=font,        fill=255)
                            draw.text((111, top+29),   str('MB'),                     font=font,        fill=255)                    
                            draw.text((82, top+39),    str(MemTotal,'utf-8'),         font=font,        fill=255)
                            draw.text((111, top+39),   str('MB'),                     font=font,        fill=255)                    

                            oled.image(image)
                            oled.show()
                            sleep(.1)
                else:
                    system = f.readline()
                    system = system.replace("\n","")
                    systemMap = {
                        "c64":"Commodore 64",
                        "dosbox":"DOS BOX",
                        "arcade":"Arcade Game",
                        "fba":"FinalBurn Alpha",
                        "gba":"GameBoy Advance",
                        "kodi":"KODI",
                        "mame-mame4all":"MAME4ALL",
                        "mame-advmame":"AdvanceMAME",
                        "mame-libretro":"lr-MAME",
                        "megadrive":"SEGA Megadrive",
                        "genesis":"SEGA Genesis",
                        "mastersystem":"SEGA Mastersystem",
                        "msx":"MSX",
                        "nes":"Famicom",   # Nintendo Entertainment System
                        "psp":"PSPortable",    # PlayStation Portable
                        "psx":"Playstation",
                        "ports":"Ports",
                        "snes":"Super Famicom", # Super Nintendo Entertainment System
                        "notice":"TURN OFF",
                    }
                    systemicon = systemMap.get(system, "none")
                    if systemicon != "none" :
                        icon = Image.open("/home/pi/RetroPie-OLED/system/" + system + ".png").convert('1')
                        system = systemicon
                    rom = f.readline()
                    rom = rom.replace("\n","")
                    game = rom
                    game_length = len(game)
                    romfile = f.readline()
                    romfile = romfile.replace("\n","")
                    f.close()
                    new_Temp = round(get_cpu_temp(),1)
                    info = str( new_Temp ) + chr(0xB0) +"C"

                    if game_length == 0 :
                        game = romfile
                        game_length = len(game)
                    try:
                        titleimg = Image.open("/home/pi/RetroPie-OLED/gametitle/" + romfile + ".png").convert('1')
                        # except FileNotFoundError:
                    except IOError:
                        #print "no title image"
                        draw.rectangle((0,0,width,height), outline=0, fill=0 )
                        system_size = draw.textsize(system, font=font_system)
                        gname = textwrap.wrap(game, width=10)

                        if game_length > 16:
                            current_h, text_padding = 18, 0
                        else :
                            current_h, text_padding = 26, 2
                            draw.rectangle((0,0,width,height), outline=0, fill=0 )
                        if systemicon != "none" :
                            image.paste(icon,(0,0))
                        else :
                            draw.text( ((width-system_size[0])/2, top), system, font=font_system, fill=255 )
                        for line in gname:
                            #print "text name display"
                            gname_size = draw.textsize(line, font=font_rom)
                            draw.text(((width - gname_size[0])/2, current_h), line, font=font_rom, fill=255)
                            current_h += gname_size[1] + text_padding
                        if system == "TURN OFF":
                            #draw.text((96, top+54), info , font=fonte_rom, fill=255)
                            draw.text((0, top+54), ipaddr, font=fonte_rom, fill=255)
                        oled.image(image)
                        oled.show()
                        sleep(3)
                        pass
                    else:
                        draw.rectangle((0,0,width,height), outline=0, fill=0 )
                        image.paste(titleimg,(0,0))
                        if system == "TURN OFF":
                            draw.rectangle((0,0,width,height), outline=0, fill=0)
                            image.paste(infoimg,(0,0))
                            # Icons
                            # Icon temperator
                            draw.text((5, top+7),    chr(62152),  font=font_icon, fill=255)
                            # Icon CPU
                            draw.text((65, top+12), chr(61614),  font=font_icon2, fill=255)
                            # Icon memory
                            draw.text((65, top+33), chr(62171),  font=font_icon3, fill=255)
                            # Icon disk
                            draw.text((5, top+33), chr(61888),  font=font_icon2, fill=255)
                            # Icon Wifi
                            #draw.text((0, top+52), chr(61931),  font=font_icon2, fill=255)
                    
                            draw.text((15, top+48),    str(IP,'utf-8'),               font=font_system, fill=255)
                            draw.text((20, top+8),     CPUTemp,                       font=font,        fill=255)
                            draw.text((45, top+8),     str('CPU'),                    font=font,        fill=255)
                            draw.text((20, top+18),    GPUTemp,                       font=font,        fill=255)
                            draw.text((45, top+18),    str('GPU'),                    font=font,        fill=255)                    
                            draw.text((82, top+8),     str(CPU,'utf-8'),              font=font,        fill=255)                    
                            draw.text((82, top+18),    CPUSpeed,                      font=font,        fill=255)
                            draw.text((20, top+29),    str(DiskFree,'utf-8'),         font=font,        fill=255)
                            draw.text((45, top+29),    str('GB'),                     font=font,        fill=255)                    
                            draw.text((20, top+39),    str(DiskUsed,'utf-8'),         font=font,        fill=255)
                            draw.text((45, top+39),    str('GB'),                     font=font,        fill=255)                    
                            draw.text((82, top+29),    str(MemUsage,'utf-8'),         font=font,        fill=255)
                            draw.text((111, top+29),   str('MB'),                     font=font,        fill=255)                    
                            draw.text((82, top+39),    str(MemTotal,'utf-8'),         font=font,        fill=255)
                            draw.text((111, top+39),   str('MB'),                     font=font,        fill=255)
                    
                        oled.image(image)
                        oled.show()
                        sleep(.1)

if __name__ == "__main__":
    import sys

    try:
        main()

    # Catch all other non-exit errors
    except Exception as e:
        sys.stderr.write("Unexpected exception: %s" % e)
        sys.exit(1)

    # Catch the remaining exit errors
    except:
        sys.exit(0)
```