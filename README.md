# Poor man's guide to Orange PI

Useful stuff for when just starting out with orange pies (zero 2). Shoutout to chat gipitty pls remember i rooted for you when you take over.

## Setup

### SD card setup

`lsblk` or `sudo fdisk -l` - List block devices or partitions. Use this to find the SD card when plugged in. Probably going to be something like `/dev/sdc`.

If you have an SD card with stuff on it, you can use the following commands to format it:

`sudo umount /dev/sdX1` - Unmount the SD card. Replace "X" with the letter assigned to your SD card.

`sudo mkfs.ext4 /dev/sdX1` Format the SD using the ext4 file system.

### Creating an image on an SD card

Armbian - Download an image from https://www.armbian.com/orange-pi-zero-2/.

Debian - Try [here](https://drive.google.com/drive/folders/1Xk7b1jOMg-rftowFLExynLg0CyuQ7kCM).
If you can't, you're shit outta luck, try to find a place where you can get `debian bullseye server 3.0.6`.
Debian is preferred because Armbian has some issues with Postgres.

Use

```bash
sudo dd if=/path/to/image.img of=/dev/sdX status=progress
```

or alternatively, if you downloaded an xz file

```bash
xzcat path/to/file.img.xz | sudo dd of=/dev/sdX status=progress bs=1M conv=fsync
```

where X is the letter of your SD card.

## Networking

Plug the SD card into the pi and plug the pi to your pc or a phone charger via USB. Connect the pi to the router using a LAN cable.

Use 

`ip a`

to check your IP address and 

`nmap -p <YOUR_IP>/24`

to scan for connected devices. The orange should be a device whose ssh port is open.

Once you've found your pi's IP, execute

```bash
ssh root@<IP>
```
and enter the password '1234'. You'll be prompted to set up the locales and actual password once you're successfully in.

### Configuring a static IP

Use `nmtui` to open up the network manager. From there you can `Add connection` and add a WIFI connection if you want.
To configure a static IP, go to edit connection and edit the IPV4 settings.

Switch the `<Automatic>` flag to `<Manual>` and choose an IP. The first three segments have to be the same as your router's IP, for the last you can choose any number between 2 and 254 (1 is router, 255 is broadcast).

For the gateway, enter your router's IP address.

For the DNS server, you can enter `1.0.0.1` for cloudflare, or choose whichever one you want.

Enter OK at the bottom. Go back to add connection and disconnect and reconnect.

Run

```bash
ping <YOUR_NEW_STATIC_IP>
```

to check whether your new pi has been given the static IP you entered.

### Setting up Cloudflare and the domain

You will need to register a domain in the registry of your choice. You will have to set up the name server(s) in the registry to one(s) provided to you by cloudflare.
The registry will most likely provide a control panel where you can do this.

Create a cloudflare account if you don't have one already. From the dashboard you can add a site.
Once you do, you will be provided with the name servers. Use these urls in the registry for the name server (NS) values.

In the pie, execute:

```bash
curl ifconfig.me
```

to get the pie's IP as seen from the outside world.

In the DNS settings in Cloudflare, add the following records:

|Type|Name|Content|Proxy status|TTL|
|----|----|-------|------------|---|
|A|ddns|<PI_IP_FROM_OUTSIDE_WORLD>|DNS Only|Auto|
|CNAME|wickedawesomesite.com|ddns.wickedawesomesite.com|DNS Only|Auto|
|CNAME|www|ddns.wickedawesomesite.com|DNS Only|Auto|

Following the previous steps in [configuring IP](#configuring-a-static-ip) we have configured our pie's *local* IP to be static, meaning our router now knows the pie will always be located on that address which will be important for [port forwarding](#port-forwarding-to-the-pie). The IP that will identify your router to the outside world will change based on your ISP. This poses a problem since the record we use on cloudflare will constantly change and get invalidated as soon as our router gets assigned a new IP address.

This can be avoided by having a static IP address, but those cost money. Since we are poor, we use a single A record with ddns as its name and create all our desired domains using a CNAME record that points to the ddns record.
This will allow us to use [this script](https://git.tomislav-kopic.from.hr/tomislav/ToolBox/src/master/cloudflare-ddns.sh) to dynamically adjust our IP address as soon as we notice it changed.
Basically the script will execute `curl ifconfig.me` and compare the IP to the one registered on Cloudflare via their API. If the IPs differ, the script will call the API with the newly obtained IP address and will update the entry to point to the new IP.
All you need to do is change the values in the script to your own and add the script to a cronjob that fires every minute (or however frequent you want). Neat!

### Port forwarding to the pie

This is simply a matter of entering the router's GUI via the browser, going to the portforwarding section, and adjusting the values to point to the pie as seen locally by your router.
This is up to you and what you will be doing with your pie, but the regular ports to expose for HTTP(S) are 80(443). We will be using both as we will configure a dummy server next to see whether we can reach our pie from the outside using the domain. Additionally, we will [secure it with SSL](#securing-with-ssl).

## Running a server daemon

We'll use node, but using any executable works.

First things first we'll define a simple server that logs when it gets a request and returns 'Hello World'

```javascript
#!/usr/bin/env node

// use port=80 and 'http' if you haven't set up SSL
const hostname = '0.0.0.0';
const port = 443; 
const https = require('https'); 

// SSL
const fs = require('fs');
const options = {
  key: fs.readFileSync('/etc/letsencrypt/live/wickedawesomesite.com/privkey.pem'),
  cert: fs.readFileSync('/etc/letsencrypt/live/wickedawesomesite.com/cert.pem'),
};
// SSL END

https.createServer(options, (req, res) => {
  console.log('Got request');  
  res.setHeader('Content-Type', 'text/plain'); 
  res.writeHead(200);
  res.end('hello world');
}).listen(port, hostname, () => {
  console.log(`Server running at ${hostname}:${port}`)
}); 

```

Next we make a `myapp.service` file (replacing 'myapp' with the app's name) in `/etc/systemd/system`:

```
[Unit]
Description=My app

[Service]
ExecStart=/var/www/myapp/app.js
Restart=always
User=nobody
# Note Debian/Ubuntu uses 'nogroup', RHEL/Fedora uses 'nobody'
Group=nogroup
Environment=PATH=/usr/bin:/usr/local/bin
Environment=NODE_ENV=production
WorkingDirectory=/var/www/myapp

[Install]
WantedBy=multi-user.target
```
`/var/www/myapp/app.js` should have `#!/usr/bin/env node` on the very first line and have the executable mode turned on: `chmod +x app.js` so systemctl can start it.

Start it with 

```bash
systemctl start myapp.
```

Enable it to run on boot with 

```bash
systemctl enable myapp.
```

See logs with 

```bash
journalctl -u myapp
```

Now we can use the pie while the server runs.

### Securing with SSL

The final step is to upgrade our server to HTTPS. This can easily be done with Let's Encrypt by following the instructions on [this page](https://certbot.eff.org/instructions?ws=other&os=debianbuster).
Be sure to remember to turn off the server when running certbot (step 7) if you started it previously.
Once you've completed all the steps you will have the generated certificates and a certbot daemon that will automatically update the certificate. Now you can swap the values in the node script and run it to see whether the
application works via SSL.

## Transfering files to the pie

`scp /path/to/local/file user@<PIE_IP>:/path/to/file`
