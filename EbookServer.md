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
