## AcuRite Weather Scripts
These scripts intercept and log the sensor data from the acurite weatherbridge, providing access to all metrics sent to acurite and weatherunderground, and optionally restricting the information/status sent back to the bridge, such as to prevent auto-update.

### Features:
The following options may be turned on/off through variables in the script.

- if an external temperature and humidity sensor is present then that data is substituted for the 5 in 1 sensors values. 
- add soil temperature to weather underground if sensor is available
- recalculate/smooth the "rainin" parameter sent to wunderground based on the 15minute accumulated rain rather than the sensor metric which resets every hour.
- store all sensor data in a sqlite database
- publish sensor data to MQTT (for use in openhab, etc....)
- log all low-level request data to/from the bridge
- freeze updates on the bridge (so this script will continue to work)
- as of March 1st, fake data is sent back to the bridge to keep it working in absense of the acurite service
- added support for the tower sensors (but you have to edit the code with your specific serial numbers)


See the "upateweatherstation.php" script for further parameters.

### Requirements:

- a webserver
- dnsmasq or equiv 

### Installation:

- edit the variables in the begining of the weathertation.php script as desired
- place all scripts inside a directory "weatherstation" on your web server directly below $HTDOCS. 
- make the ip address of the weatherbridge static by using dhcp reservations
- fake entries (dnsmasq, etc...) for hubapi.myacurite.com and rtupdate.wunderground.com so that your webserver receives the weatherbridge GET instead of the real DNS hosts.

One easy way to add dnsmasq entries is with a router that supports it  (dd-wrt, tomato, merlin, etc...), for example (assuming 192.168.1.250 is the acurite bridge) add the following to /tmp/etc/hosts and /jffs/config/hosts.add on the router:	

	192.168.1.250 mylocalwebserver hubapi.myacurite.com rtupdate.wunderground.com

### Testing:

If all is working, and logging is enabled,  you should see the following files under $HTDOCS/weatherstation:

- **logfile.txt** logging information
- **weather.dat** key-values containing the most recent metrics and values
- **weather.db3** database, if you have enabled the database (use something like *sqlite3 weather.db3*)


### Notes on Apache Configuration:

The configuration for the weatherstation directory should include something like the following (based on modrewrite) because acurite does not use a script extension on their URI. 

	RewriteCond %{REQUEST_FILENAME} !-d
	RewriteCond %{REQUEST_FILENAME}\.php -f
	RewriteRule ^(.*)$ $1.php

The weatherbridge uses http so you need to have Apache listening on port 80. If running with SSL remember to disable it for the weatherstation script, and also consider not logging the constant traffic.

	RewriteRule "^/weatherstation" - [L]
	# other rules that redirect traffic to 443

        # don't log the constant bridge traffic 
        SetEnvIf Request_URI "^/weatherstation.*$" dontlog
        CustomLog ${APACHE_LOG_DIR}/access.log combined env=!dontlog

### Notes on Nginx (as reverse proxy to apache server) Configuration:

```
server {
        listen                  80;
        server_name             hubapi.myacurite.com;
        server_tokens           off;
        proxy_buffering         off;
        location / {
          if (!-e $request_filename){
              rewrite ^(.*)$ /$1.php;
          }
          proxy_pass http://$location of apache php server$:$port$;
          }
        }
```

(any help documenting how to get nginx working as php server welcome)


### How to flash the memory of the Smarthub to update Wunderground settings
  Wunderground made a change from passwords to keys for personal weather station updates.  As of August 2019 it is still possible to send http updates to rtupdate.wunderground.com directly from the Smarthub, but without the correct password (now your "key") it won't work.  You can make changes to the Smarthub memory once your server is up and functioning correctly by temporarily modifying the line towards the end of the updateweatherstation.php file in the following way:

  ```
        echo "{\"localtime\":"."\"$timestr\",\"checkversion\":\"224\",\"ID1\":\"$YOUR_PSW_ID$\",\"PASSWORD1\":\"$YOUR_KEY\",\"sensor1\":\"$8_DIGIT_ID_OF_PWS$\",\"elevation\":\"$ELEVATION_IN_FT$\"}";
  ```

  save the file, the hub will soon be getting json responses to update memory values. Revert to the original:

  ```
        echo "{\"localtime\":"."\"$timestr\",\"checkversion\":\"224\"}";
  ```

Wait a few seconds and save again. This will resume sending the default json updates.

Note: Wireshark is your friend to see whats going on. 

  If your device is now sending the correct http string you can take away the rtupdate.wunderground.com entry in DNSmasq/resolver; etc/hosts file of your router...

Note:  It is important to fill in the correct elevation of your hub to get accurate readings for your barometric pressure.


### Debugging:
Normal messages in the log  (LOGRESPONSE=true) look like:

	[2017/03/17 00:19:01] acurite_from: {"localtime":"00:00:00","checkversion":"224"}
	[2017/03/17 00:19:01] acurite_brdg: {"localtime":"00:00:00","checkversion":"224"}
	[2017/03/17 00:19:55] wunderground: success

There are two acurite GET requests from the bridge for every weatherunderground GET. The "localtime" parameter does not update in realtime, and is sometimes absent. All acurite responses are json.

Wunderground will return the string "success" mostly, and occasionally return garbage or timeout. These errors should be expected.

	[2017/03/15 04:50:33] Curl error: rtupdate.wunderground.com:52.25.21.79: Empty reply from server

### Openhab Samples:
The directory openhab contains a sample configuration using the MQTT bindings.

![alt text](screenshots/image1.jpg)
![alt text](screenshots/image2.jpg)
