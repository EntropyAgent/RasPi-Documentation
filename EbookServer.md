# Set up RasPi Zero as a standalone COPS Ebook Server.

##### Inital Setup
1. Download Rasbian Lite and write to SD Card.
2. Mount your SD Card on your computer and open the Boot directory.
   - Make a file called ssh in boot `touch ./ssh`
   - In `config.txt` add `dtoverlay=dwc2`
   - in `cmdline.txt` after rootwaid add `modules-load=dwc2,g_ether g_ether.host_addr=36:7c:9b:c8:63:00` this will enable OTG ethernet adaptor and assign a static mac address for the adapter.
3. Put your SD card into your Pi and boot it up. Connect a USB cable from the Pi to your computer. At this point your computer should detect a new network adapter. You need to configure that adaptor for "Link Local"  connection type and share internet to it if you can. That will allow you to connect the Pi to the internet for downloading updates.
4. Connect to the Pi using ssh `ssh pi@raspberry.local`
5. Configure some stuff in `raspi-config`
   - Expand File System
   - Set Timezone
   - Set Hostname
6. Update and upgrade the system `sudo apt update` the `sudo apt upgrade`
7. Reboot

##### Setup WiFi Access Point
1. Install hostapd and dnsmasq. `sudo apt install hostapd dnsmasq`
2. Stop newly installed services until they are configured properly.
   - `sudo systemctl stop dnsmasq`
   - `sudo systemctl stop hostapd`
3. We have to configrue as static IP on the wireless interface so our server is reachable. This documentation assumes that you want to use a standard 192.168.x.x IP address for the Pi. This should work in almost all cases. Also we are assuming you want to use `wlan0` as your wireless interface.
   - Edit the dhcpcd configuration file:
      `sudo nano /etc/dhcpcd.conf`
      Go to the end of the file and add:
      ```
      interface wlan0
         static ip_address=192.168.4.1/24
      ```
      Now restart the dhcpcd daemon `sudo service dhcpcd restart`
2. Now we need to configure the DHCP server (dnsmasq).
   - The default configuration contains a ton of extra options that we don't want to use so we will backup the default configuration file `sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig`
   - Then we will create a new one. `sudo nano /etc/dnsmasq.conf`
      Type or copy the following into the `dnsmasq.conf` file and save it.
      ```
      interface=wlan0
         dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
      ```
      So for `wlan0`, we are going to provide IP addresses in the range of 192.168.4.2 to .20, with a lease time of 24 hours.
3. Now lets configure the access point software (hostapd)
   - Edit the hostapd configuration file. `sudo nano /etc/hostapd/hostapd.conf`
      Add the following to the file:
      ```
      interface=wlan0
      driver=nl80211
      ssid=BookServer   # SSID Name
      hw_mode=g
      channel=7
      wmm_enabled=0
      macaddr_acl=0
      auth_algs=1
      ingnore_broadcast_ssid=0
      wpa=2
      wpa_passphrase=YourPassword   # SSID Password
      wpa_key_mgmt=WPA-PSK
      wpa_pairwise=TKIP
      rsn_pairwise=CCMP
      ```
   - Now we need to tell hostapd where to find this file: `sudo nano /etc/default/hostapd`
      Find the line with `#DAEMON_CONF` replace it with this:
      `DAEMON_CONF="/etc/hostapd/hostapd.conf`
4. Start your services!
   - `sudo service hostapd start`
   - `sudo service dnsmasq start`
   - Make sure there are no errors. If all is good reboot your Pi and disconnect it from the computer. `sudo reboot`
5. Connect to your new "BookServer" access point and make sure you have SSH access. `ssh pi@raspberry.local`

##### Install Web Server PHP and COPS
1. Install all the reqired packages.
   - `sudo apt install nginx php-fpm php7.0-gd php7.0-sqlite3 php7.0-json php7.0-intl php7.0-xml php7.0-mbstring php7.0-zip`
2. Test the Web Server
   - Open a browser and go to http://raspberry.local You should see the default nginx page. This is located at /var/www/html and is where any web content you want to serv has to be located.
3. Edit the nginx config file to enable PHP
   - Open the config file `sudo nano /etc/nginx/sites-enabled/default`
   - Roughly around line 25 you will see a line that says `index index.html index.htm;`
      Add `index.php` to that line. It should look like `index index.php index.html index.htm;` when you are done.
   - Scroll down until you see a line that looks like:
      ```
      # pass the PHP scripts to FastCGI server listening on
      127.0.0.1:9000
      #
      # location ~ \.php$ {
      ```
      Edit by removing the `#` characters on the following lines:
      ```
      location ~ \.php$ {
         include snippets/fastcgi-php.conf;
         fastcgi_pass unix:/var/run/php7-fpm.sock;
      }
      ```
      It should now look like this:
      ``` 
      pass the PHP scripts to FastCGI server listening on
      127.0.0.1:9000
      #
      location ~ \.php$ {
         include snippets/fastcgi-php.conf;
         fastcgi_pass unix:/var/run/php7-fpm.sock;
      }
      ```
      Note: Depending on certian versions of components and of Raspbian. Yours may look slightly different.
   - Reload the configuration file. `sudo /etc/init.d/nginx reload`
4. Test PHP
   - Delete `index.nginx-debian.html` and create `index.php`
      ```
      cd /var/www/html/
      sudo rm index.nginx-debian.html
      sudo touch index.php
      ```
   - Add some dynamic PHP content to our index file to test PHP.
      `sudo nano index.php`
      Add `<?php echo phpinfo(); ?>` to that file and save it.
   - Refresh your browser or go back to http://raspberrypi.local and make sure you see PHP info.
