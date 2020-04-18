# So you want to have a Little Printer

Good news! You came to the right place. Just follow these ??? easy steps and you'll be set up in no time* (a fair amount of time, actually.)

# What you'll need

* A thermal printer, ideally a Paperang P1. There are a number of places where you can buy one: Amazon ([US](https://www.amazon.com/PAPERANG-Portable-Language-Wireless-Bluetooth/dp/B07TCJ2TC6/), [UK](https://www.amazon.co.uk/PAPERANG-Wireless-Mobile-Instant-Printer/dp/B078HZK2NV), [DE](https://www.amazon.de/Paperang-Mini-Drucker-Android-Ger%C3%A4te-Druckpapier-gew%C3%B6hnlich/dp/B082PWV57H/)), [Thepaperang](https://thepaperang.com/products/paperang-p1), [Paperangprint](https://paperangprint.com/collections/paperang-printers/products/paperang-p1), [Aliexpress](https://www.aliexpress.com/w/wholesale-paperang-p1.html) and so on. Don't forget to buy a few rolls of paper for it; you can find them in many colors, and sticker paper is especially fun!
* Ideally, a Raspberry Pi with Bluetooth, but we've tested this with a Mac as well, and with some tweaks it will work on any computer with Bluetooth running *nix
* Node.js 10, Python 3.7, Docker

# How to get started

We'll assume you have your Paperang. Great! You can already use via its app on your smartphone, but that's not what we'll do.

Connect your Paperang to a power source with a USB cable to prevent drain. It should also keep it turned on. You can communicate with the printer via USB but for now, we'll just use bluetooth.

Pair your Paperang with your computer over Bluetooth and make sure you have Node 10, Python 3.7 and Docker installed.

# Let's make a (fake) printer

This project is predicated on us prentending to be a Little Printer, so let's do that.

Clone the `sirius` repository from github: `git clone https://github.com/nordprojects/sirius`, install the required packages (`pip3 install -r requirements.txt`) which does not make it work right now so do a `docker-compose up` and get a cup of coffee.

Once it's up and running, ssh into ??? and run `./manage.py fake printer`, which generates a `*.printer` file, it looks like this:

```
  address: abcdef0123456789
       DB id: 8
      secret: 4a8489a9b8
  claim code: 123a-456b-789c-012d
```

Next, we're going to claim the printer on the nord-sirious instance. Go to [https://littleprinter.nordprojects.co/](https://littleprinter.nordprojects.co/), sign in with Twitter, and click "Claim a printer". Enter the claim code you have and give it a name, then hit "Claim Printer".

Congratulations, you now have a fake Little Printer!

# Wiring it all up

Next, we'll put it all together, connecting our Paperang to the network. For that, we use two projects: [sirius-client] and [python-paperang]. They should really be merged into one. One day.

Anyways, clone both of them from Github, and we'll start with some testing with the latter project. We're going to assume you have at least a passing understanding of both Python and Typescript.

If you're on linux, and we'll assume Debian, you'll need the following packages installed: `libbluetooth-dev libhidapi-dev libatlas-base-dev python3-llvmlite python3-numba python-llvmlite llvm-dev`








# How the whole fake printer thing works (by Josh)

Josh: It takes the printer ID (mac address of printer iirc, that you can generate) and makes a random 16 digit code ("claim code"). From there, it posts that to the server as part of the normal handshake. Then the device just...waits. eventually the user logs onto the server, types in claim code, then you own that printer ID, and the server will start firing payloads at it. It's all very....simple. no like, fancy key exchange, or identification, or anything.

KTamas: right, so if i want to make a new printer, what do i do?...

Josh: so, the most reliable way to make a printer is to clone `https://github.com/nordprojects/sirius` then run `./manage.py fake printer` -> generates `*.printer` file. Looks like:

```
  address: abcdef0123456789
       DB id: 8
      secret: 4a8489a9b8
  claim code: 123a-456b-789c-012d
```

Sso that is "a printer", essentially. but what do we do with that? ... well! When we connect to the server (via websocket), we don't need to send any special "it's my first time" payload, we just send a regular payload every time, that looks like:

```
{
    type: 'BridgeEvent',
    json_payload: {
      ncp_version: '0x46C5',
      uptime: '45.71 23.94',
      firmware_version: 'v2.3.1-f3c7946',
      network_info: {
        extended_pan_id: '0xredacted',
        node_type: 'EMBER_COORDINATOR',
        radio_power_mode: 'EMBER_TX_POWER_MODE_BOOST',
        security_level: 5,
        network_status: 'EMBER_JOINED_NETWORK',
        channel: 11,
        security_profile: 'Custom',
        power: 8,
        node_eui64: '0xredacted',
        pan_id: '0xDF3A',
        node_id: '0x0000',
      },
      name: 'power_on',
      local_ip_address: '192.168.1.98',
      uboot_environment:
        'long text',
      mac_address: 'redacted',
      model: 'A',
    },
    bridge_address: 'redacted',
    timestamp: 1426256447.70695,
  }
```

(Much of that can actually be ignored I think - sirius isn't using it - but that's what it looks like.) So that gets the bridge online, not the printer. To get the printer online, it sends a packet afterward that looks like:

```
{
    timestamp: 1419107228.91187,
    type: 'BridgeEvent',
    json_payload: {
      name: 'encryption_key_required',
      device_address: 'redacted',
    },
    bridge_address: 'redacted',
  }
```
So at that point, the server knows about that bridge, and now it knows the printer on that bridge. That's all the client does. on to the server claim code! Tong story short, it mostly lives here:
https://github.com/nordprojects/sirius/blob/master/sirius/coding/claiming.py. So when you type a claim code into the website, it gets unpacked to find the bridge ID and the device ID: https://github.com/nordprojects/sirius/blob/master/sirius/coding/claiming.py#L70-L89 and then it associates that device to your account. voila! THE GOOD NEWS IS that in searching for that, I found an actually documented summary of what claim codes are: https://github.com/nordprojects/sirius#claim-codes.

KTamas: and after all this, it appears permanently on device.li?

Josh: Yurp, essentially. I guess technically "until someone else claims it", if the device is reset etc. but I don't know if that regenerates the ID, or if it's truly based on the mac address I guess it could be easy enough to make a lil microsite that generates new addresses + claim codes, to make this part easier.