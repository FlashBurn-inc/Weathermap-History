# What is this?

This repo is just the web html to display the animated weathermap GIFs based on chosen date.

Please follow the guide below, which will walk you through the shell scripts which create GIF animations of the weathermap daily.

---

## Prerequisites
- "convert" command (yum install ImageMagick , and on ubuntu apt install imagemagick ) - used to convert pngs to gif
- "pngquant" command (yum install pngquant , apt install pngquant) - PNG Optimization
- "gifsicle" command (yum install gifsicle, apt install gifsicle) - GIF Optimization which shows a GIF filesize reduction of 87%
- Assumes your librenms path is /opt/librenms (otherwise you will need to edit the scripts)

---

##  How do I use this?

**mkdir -p /opt/weathermap-history/history**
**cd /opt/weathermap-history**

**vi getweather.sh** (edit file path if wrong)
script collects all existed maps from Weathermap output folder
```
#!/bin/bash
DATE=`date '+%y%m%d%H%M'`
for map in $(ls -l /opt/librenms/html/plugins/Weathermap/output/ | grep png | awk '{print $9}' | awk -F\. '{print $1}');do
cp /opt/librenms/html/plugins/Weathermap/output/$map.png /opt/weathermap-history/history/${map}_$DATE.png
done

```
**chmod +x getweather.sh**
**chown librenms:librenms getweather.sh** (change to librenms user/group)

**vi makeGIF.sh** (edit file path if wrong)
script makes yesterday history gif files for all maps 
```
#!/bin/bash

DATE=$(date "+%Y-%m-%d" -d "yesterday")
PATH_DIR=/opt/librenms/html/plugins/Weathermap/output
cd /opt/weathermap-history/history/
pngquant --force --quality=60-70 *.png -f --ext .png
for map in $(ls -l $PATH_DIR | grep png | awk '{print $9}' | awk -F\.png '{print $1}');do
mkdir -p $PATH_DIR/history/$map/$DATE
convert  -delay 50 -loop 0 ${map}_*.png $PATH_DIR/history/$map/$DATE/$DATE.gif
gifsicle -O3 $PATH_DIR/history/$map/$DATE/$DATE.gif -o $PATH_DIR/history/$map/$DATE/$DATE.gif
done
JS_ARRAY=$(ls -l $PATH_DIR | grep png | awk '{print $9}' | awk -F\.png '{print "!"$1"!,"}' | xargs | sed 's/!/\x27/g' | sed 's/,$//g')
sed -i '15d' $PATH_DIR/history/index.html
sed -i '14a\var arr = new Array('"${JS_ARRAY}"');\' $PATH_DIR/history/index.html

rm /opt/weathermap-history/history/*
```
**chmod +x makeGIF.sh**
**chown librenms:librenms makeGIF.sh** (change to librenms user/group)


**vi makeGIF_today.sh** (edit file path if wrong)
script makes temporary today history files for all maps 
```
#!/bin/bash

DATE=$(date "+%Y-%m-%d")
PATH_DIR=/opt/librenms/html/plugins/Weathermap/output
cd /opt/weathermap-history/history/
pngquant --force --quality=60-70 *.png -f --ext .png
for map in $(ls -l $PATH_DIR | grep png | awk '{print $9}' | awk -F\. '{print $1}');do
mkdir -p $PATH_DIR/history/$map/$DATE
convert  -delay 50 -loop 0 ${map}_*.png $PATH_DIR/history/$map/$DATE/$DATE.gif
gifsicle -O3 $PATH_DIR/history/$map/$DATE/$DATE.gif -o $PATH_DIR/history/$map/$DATE/$DATE.gif
done
JS_ARRAY=$(ls -l $PATH_DIR | grep png | awk '{print $9}' | awk -F\.png '{print "!"$1"!,"}' | xargs | sed 's/!/\x27/g' | sed 's/,$//g')
sed -i '15d' $PATH_DIR/history/index.html
sed -i '14a\var arr = new Array('"${JS_ARRAY}"');\' $PATH_DIR/history/index.html
```
**chmod +x makeGIF_today.sh**
**chown librenms:librenms makeGIF_today.sh** (change to librenms user/group)


This sting will add a link to history maps to the submenu of the Weathermap plugin
**vi /opt/librenms/html/plugins/Weathermap/Weathermap.php** (edit file path if wrong)
and add string 
```
$submenu .= ' <li><a href="/plugins/' . self::$name . '/output/history/index.html' . '"><i class="fa fa-folder fa-fw fa-lg" aria-hidden="true"></i> '. History . '</a></li>';
```
after these strings
```
        //Create submenu
        $submenu = ' <ul class="dropdown-menu scrollable-menu">';
        $count = 0;
        
```

Open librenms cronjob

**vi /etc/cron.d/librenms**

(The first line will create PNGs at 5 minute intervals between 19:00 up to 23:55, change this to your busiest periods but i would keep up to a 5 hour period otherwise you end up with huge file sized GIFs)  
(The second line creates the temporary GIF every hours during the day)
(The third line creates the GIF at 3am, I would leave this because we explicitly set -d yesterday to compensate the date)

```
# weathermap-history
*/5 19-23 * * * librenmswww /opt/weathermap-history/getweather.sh >> /dev/null 2>&1
55 19-23 * * * librenmswww /opt/weathermap-history/makeGIF_today.sh >> /dev/null 2>&1
0 3 * * * librenmswww /opt/weathermap-history/makeGIF.sh >> /dev/null 2>&1
```

**cd /opt/librenms/html/plugins/Weathermap/output**


Now clone the repo into the history folder like so
```
git clone https://github.com/chasgames/Weathermap-History history
```

In evening you should have the date folder and GIF created in this folder opt/librenms/html/plugins/Weathermap/output/history/

You can go to the web front end here:
http://1.1.1.1/plugins/Weathermap/output/history/index.html

---

##  Troubleshooting

When selecting a date, GIF doesn't show -
Depending on GIF size it can take a while to load, 1M < GIF should take less than 10 seconds. 5M GIF can take up to 30 seconds so you will need to shorten the time period or use the resize function in gifsicle. Use chrome inspect tool to find out the GIF link it's trying to pull and check it exists on the filesystem.

GIF is not made -
Try running ./getweather.sh manually it should copy the weathermap into /opt/weathermap-history/history as a temporary location to hold the pngs for the time period. Check that works first. Then redirect the cronjobs to a log file instead of /dev/null to see what the problem is.


---

##  Acknowledgments
- https://github.com/CaptainCodeman/gif-player (MIT)
