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
