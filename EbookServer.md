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
      ssid=NameOfNetwork
      hw_mode=g
      channel=7
      wmm_enabled=0
      macaddr_acl=0
      auth_algs=1
      ignore_broadcast_ssid=0
      wpa=2
      wpa_passphrase=AardvarkBadgerHedgehog
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
5. Install COPS
   - Make sure you are in your home directory `cd ~`
   - Create a folder in your home directory for the Calibre library to be mounted to. `mkdir /home/pi/library`
   - Download and unzip COPS:
      ```
      wget https://github.com/seblucas/cops/releases/download/1.1.1/cops-1.1.1.zip
      mkdir cops
      mv ./cops-1.1.1.zip ./cops
      cd ./cops
      unzip cops-1.1.1.zip
      ```
   - Move COPS to our /var/www/html directory so nginx can see it:
      ```
      cd ~
      sudo mv ./cops /var/www/html/
      ```
   - Setup the COPS configuration file
      ```
      cd /var/www/html/cops
      cp config_local.php.example config_local.php
      ```
   - Edit the configuration file so `$config['calibre_directory'] = './';` looks like `'$config['calibre_directory'] = '/home/pi/library/'` Be certain you have the trailing /
   - Go to http://raspberrypi.local/cops/ You should see a config check page for COPS complaining that it cannot find the Calibre database file. That is fine. We are going to fix that next.
   
##### Set up USB Mass Storage Mode
1. First we have to remove the OTG Ethernet driver we installed initially. We will still be able to communicate with the PI via its WiFi that it is broadcasting.
   - Change directory to /boot/ and edit the `cmdline.txt` file. Remove `g_ether.host_addr=36:7c:9b:c8:63:00`
   - Reboot the Pi and reconnect to the WiFi
2. The USB device that your computer will see is actually a file that is on the Rasberry Pi. We need to create that file and isntall a file system on it.
   - Create a 2GB file in your home directory called piusb.bin
      ```
      dd if=/dev/zero of=./piusb.bin bs=1M count=2000
      ```
      This will take a long time to complete. It is creating a file filled with zeros that is 2 gigabytes in size.
   - Next we have to put a filesystem into that file. The recommeded file system for a USB stick is FAT32 or VFAT.
      `mkfs.vfat ./piusb.bin -n Calibre`
3. Now we have a file that will mount to your computer when the Pi is plugged in. We just need to enable the USB mass storage mode.
   - Open `/etc/rc.local` in nano and go to the bottom. Before the `exit 0` line add:
      ```
      # Setup USB Mass Storage Stuff
      losetup /dev/loop0 /home/pi/piusb.bin
      mount -o ro,uid=1000,gid=1000 /dev/loop0 /home/pi/library
      modprobe g_mass_storage file=/dev/loop0 removeable=1 ro=0 stall=0
      ```
      This will cause losetup to mount your image file to a loop device, mount will mount that loop device to your COPS library directory, then modprobe will load the mass_storage kernel module and activate it. This will happen at boot every time.
   - Power off your Pi. `sudo poweroff`
   - Disconnect the power cable from your Pi if you are using one. Connect a cable from the USB on your computer to the USB on the Pi. This will power the Pi and also allow your computer to see the Pi as a USB device.
   
##### Load your Calibre library onto the Pi
1. Boot up your Pi and wait for your computer to see the USB drive. Once it is mounted Copy your entire Calibre directory to the drive. It may take a while depending on the size of your directory. Once done copying you should be able to go to http://raspberrypi.local/cops/ and see your books. 
