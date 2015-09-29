# Roomie <-> cec-web bridge
This project provides property lists describing devices that can be used with [Roomie Remote][Roomie] (a home theater remote control for iOS) in tandem with [cec-web][cec-web] (RESTful webservice to control devices via the CEC bus in HDMI)

# Why?
I love most everything about Roomie. One of the only things that I don't like is that they don't have 1000 developers working 24/7 to reverse engineer arcane control protocols for all my devices, like the Sony PlayStation 4 or the Apple TV (although the recently got great support for the latter!). After purchasing a PS4, I became annoyed that I couldn't do simple things like turn it on and off remotely through Roomie.

So, I started looking around for a solution and found a [post on the Roomie forums][jasmas-post] by [jasmas][jasmas], explaining that he had set up a bridge between Roomie and [HDMI-CEC][wphdmi] through his Raspberry Pi to control devices in his setup that didn't have native Roomie support. I realized that this was a great solution, and I already had a Pi hooked up to my TV serving as a Plex client anyway! I quickly cloned his project, [cecd][jasmascecd] and with a few minutes of work was amazed that I was controlling my TV through Roomie, and it was pretty fast! However, I was hungry for more. I forked his project into [my own cecd][robbiescecd] and quickly wasted a few hours learning all about CEC and building a bigger property list with more commands added. I was pretty happy, as I was controlling my TV and now my PS4 with Roomie and it worked pretty well.

After using this setup for a few months now, I had found quite a few issues with it. Namely, it's really a giant hack that pipes TCP input into [socat][socat] which then sends it onto cec-client from [the libcec project][libcec]. I would have to restart my scripts every few days, it was sometimes prone to lag, and just wasn't a clean solution overall. So I started looking for a new solution. Initially, I tried using [adammw's][adammw] [node-cec][node-cec] since Node.js is my favorite language (yay Javascript!) and spent a few weeks banging my head against the wall, trying to read C code and understand how to make things work. It proved fruitless so I quickly gave up. In addition, node doesn't seem to run so well on Raspberry Pi and requires work to set it up and get the modules installed. 

I started looking around for a new solution and stumbled onto [cec.go][cec.go] by [chbmuc][chbmuc]. This looks great! I haven't done much work with Go except a very short stint years ago when it was very young, but I'll give it a try. I kept banging my head against the wall and made some good progress on a Go implementation of a TCP server that is connected to cec.go, but I realized that it wouldn't work that well, and I could make life a lot easier. As I was looking around again for ideas, I finally found my answer to all my problems: [cec-web][cec-web] another project by [chbmuc][chbmuc]. Finally! Someone did some of my work for me! Roomie knows how to speak HTTP, so this was a match made in heaven. Within a few minutes I was up and running with Go on my Raspberry Pi and compiled cec-web.go. It was so much faster! Everything worked! Exclamation points! I quickly wrote up a property list with all of the supported CEC commands (thankfully, he already had wrote a map of human readable commands to hex values). After a few hours troubleshooting and coming up with a perfect property list, I had it. A fully working, generic implementation of a property list covering all of the codes, including some edge cases and extra functionality (like HDMI input switching). I was happy, Roomie was happy, my PS4 was happy since it wasn't being left on accidentially all the time anymore! So here we are... (also, why did I just write this novel in a README file. I guess it's just the excitement levels)

# Instructions

## Requirements

You need a few things to get started:
- A Raspberry Pi, or a computer, connected to your TV/amplifier/your output source, with a CEC adapter. The Raspberry Pi has one built in already but if you don't want that you can get one from [Pulse-Eight][usb-cec] that connects via USB. I have one for my dedicated HTPC and it works really well!
- [Go][golang] needs to be installed on your device of choice. For Raspberry Pi, you can get builds from [Dave Cheney's website][go-pi]. 
- Roomie Remote of course! I don't think you need anything more then the free app, but you do need to have it setup to sync to a destination you can control. I use Dropbox and Roomie Agent, but when building property lists I just use Dropbox
- A Mac, or something that you can edit property lists on. Yes, property lists are just XML, but Roomie likes to convert the property lists to binary format when you import them. You may need to run `plutil convert -xml1 RoomieCodes.plist` in terminal to convert them from binary to text format.

## Caveats
- The device you want to control must support HDMI-CEC. While CEC is a standard, many manufacturers choose to implement it in their own way, so one command may work differently or not at all between two devices. You also need to have CEC support enabled on your devices. No one seems to call CEC by it's actual name, so if you are looking around for the setting to enable it and can't find it, there is a [handy list of each manufacturer's trade name for HDMI-CEC](#hdmi-cec-trade-name-list). You should only need to turn it on.

## Steps to get running with the Master Code list
1. Make sure you meet all the requirements
2. Download and build my [cec-web][robbiet480-cec-web] fork
  1. Clone the repository
  2. Run `go get`
  3. Run `go build cec-web.go`
3. Run cec-web: `./cec-web -i 0.0.0.0 -n "Roomie CEC-Web"` (you can change the OSD name to anything you want. Also, you may have faster startup times if you add `-a RPI` to the end of the command, assuming you are using a Raspberry Pi)
4. Clone this repository
5. Copy MasterCodes-RoomieCodes.property list to the folder that Roomie syncs to/from. For me, it's at `~/Dropbox/Roomie`.
6. Rename it to RoomieCodes.property list
7. Open Roomie on your device and restore from Dropbox, or use the Roomie Agent to restore the configuration
8. Now, you'll be able to add a new device. The master codes list is set to appear as an Auxiliary device with a manufacturer name of cec-web.
9. After selecting the All Options device, Roomie will ask you for the IP address and Port number to use. By default, `cec-web` starts on port `8080`.
10. After pressing done, you'll see the device screen. You need to go all the way down to the bottom and put in the device identifier for the device you want to control. See the section about [finding your device identifier](#finding-your-device-identifiers) below.
10. Hit Done and test it out! Everything should work!

## Steps to make your own code sets
I've added two of my own code sets in this repository already, one for a Sony PlayStation 4 and one for my Toshiba 55WX800U TV. You can use them as base configurations and customize them to your needs. These instructions are written for creating a code set for PS4, but the same logic can be applied for any device type.

1. Copy MasterCodes-RoomieCodes.property list from the top level of this repository into the folder that best describes your device type.
2. Rename it to your device name/model. I named mine PlayStation 4.property list.
3. Open the file for editing.
4. Let's start with the easy stuff. At the bottom of the file, you'll find the information describing the device. `brand` is the manufacturer name, `cat` is the name/model of the device and `type` corresponds to a Roomie Type (listed below). Do not modify `method`. I set `brand` to Sony, `cat` to PlayStation 4 and `type` to 8 which is Game System. To find the correct code number, refer to the [Roomie DDK][roomie-ddk]
5. Now for the fun part, figuring out your codes. The first thing to do is remove all the codes that just don't make sense for your device. You wouldn't add input changing commands to a Roku right? The only ones that I left for my PS4 commands were the cursor commands, power commands, menu commands and eject. You can also name the commands whatever you want, but you should make sure to use Roomie DDK recommended commands for best integration with Roomie.
6. Save the file
7. Assuming this is your only custom device, copy it to your Roomie folder and rename it RoomieCodes.property list.
8. Import it to Roomie

## Adding custom CEC commands
All of the commands that are in the MasterCodes file are for the User Button Pressed function in the CEC spec, but you can also add raw CEC commands that will be passed through directly. You can see that's how I build the INPUT HDMI commands out. Just copy the same format into a new command. You can also of course add any of the other endpoints that [cec-web]() has to offer.

As an alternative to building your own code set entirely, you can use `.KEY SEND` to send a custom key code that may not be listed in MasterCodes, i.e. if there is a vendor specific code that does what you need, you can use `.KEY SEND` and put the correct hex code in as the parameter. The valid hex codes are listed in the [HDMI-CEC specs][cec-specs] on page 95 (titled _CEC Table 27 User Control Codes_). You can also generate them from [CEC-O-Matic][cec-o-matic].

## Finding your device identifiers
In the code sets, `$AUTH1$` is used to allow you to use one generic code set for any device type. It expects a device identifier to be given as the authentication info (although this is really just a hack, as we don't use it for auth)
To get a list of all of your devices and their identifiers:
  1. Make sure that `cec-web` is running on your CEC-enabled device and is accessible from another computer on your network.
  2. Go to `http://192.168.1.2:8080/info`, where 192.168.1.2 is the IP address of the device running `cec-web`
  3. A JSON blob will be returned to you, describing all of the connected/addressable devices. Your device identifier is the key of the object that describes the device you want. For example, if I have this JSON blob:
```
{
    "Playback": {
        "OSDName": "Chromecast",
        "Vendor": "",
        "LogicalAddress": 4,
        "ActiveSource": false,
        "PowerStatus": "on",
        "PhysicalAddress": "1.0.0.0"
    },
    "Playback2": {
        "OSDName": "PlayStation 4",
        "Vendor": "Sony",
        "LogicalAddress": 8,
        "ActiveSource": false,
        "PowerStatus": "standby",
        "PhysicalAddress": "3.0.0.0"
    },
    "TV": {
        "OSDName": "TV",
        "Vendor": "Toshiba",
        "LogicalAddress": 0,
        "ActiveSource": false,
        "PowerStatus": "standby",
        "PhysicalAddress": "0.0.0.0"
    },
    "Tuner": {
        "OSDName": "cec-web",
        "Vendor": "Toshiba",
        "LogicalAddress": 3,
        "ActiveSource": false,
        "PowerStatus": "on",
        "PhysicalAddress": "2.0.0.0"
    }
}
```
  and I want to control my PS4, the device identifier i'd use is `Playback2`. If you don't normally look at JSON all day like I do, it may be helpful to copy the entire blob and paste it into [JSONLint][jsonlint] for better readability.


# Contributing
If you create your own code sets, *please* contribute back! Just submit a pull request and i'll get right to merging it.

# Thanks
This wouldn't have been possible without [jasmas][jasmas], [chbmuc][chbmuc], [adammw][adammw] and of course the hard work of everyone at Roomie Remote. 

# License
MIT

# Resources
[cec-web][cec-web]

[cec.go][cec.go]

[libcec][libcec]

[CEC-O-Matic][cec-o-matic] - A great way to generate valid CEC commands

[The HDMI-CEC specification PDF][cec-specs]

[The Roomie Remote Device Development Kit (DDK)][roomie-ddk]

# HDMI-CEC Trade Name list
- Anynet+ (Samsung)
- Aquos Link (Sharp)
- BRAVIA Link and BRAVIA Sync (Sony)
- HDMI-CEC (Hitachi)
- E-link (AOC)
- Kuro Link (Pioneer)
- INlink (Insignia)
- CE-Link and Regza Link (Toshiba)
- RIHD (Remote Interactive over HDMI) (Onkyo)
- RuncoLink (Runco International)
- SimpLink (LG)
- T-Link (ITT)
- HDAVI Control
- EZ-Sync
- VIERA Link (Panasonic)
- EasyLink (Philips)
- NetCommand for HDMI (Mitsubishi)

# Roomie Device Type list
| ID     | Name                | Notes                  |
|--------|---------------------|------------------------|
| 0      | A/V System          |                        |
| 1      | Auxiliary           |                        |
| 2      | Blu-ray Player      |                        |
| ~~3~~  | Cable               | _(Deprecated, use 18)_ |
| 4      | CD                  |                        |
| 5      | DTV Converter       |                        |
| 6      | DVD                 |                        |
| 7      | DVD/VCR Combo       |                        |
| 8      | Game System         |                        |
| 9      | Home Theater System |                        |
| 10     | iPod Dock           |                        |
| 11     | LaserDisc           |                        |
| 12     | Lighting            |                        |
| 13     | MediaPlayer         |                        |
| 14     | Multizone System    |                        |
| 15     | Projector           |                        |
| 16     | Receiver/Pre-Amp    |                        |
| ~~17~~ | ~~Satellite~~       | _(Deprecated, use 18)_ |
| 18     | Set Top Box         |                        |
| 19     | Soundbar            |                        |
| 20     | Subwoofer           |                        |
| 21     | Switcher            |                        |
| 22     | Tuner               |                        |
| 23     | TV                  |                        |
| 24     | TV/DVD Combo        |                        |
| 25     | TV/DVD/VCR Combo    |                        |
| 26     | TV/VCR Combo        |                        |
| 27     | VCR                 |                        |
| 28     | Video Processor     |                        |
| 29     | Climate Control     |                        |
| 30     | Video Camera        |                        |

[Roomie]: http://roomieremote.com
[cec-web]: https://github.com/chbmuc/cec-web
[robbiet480-cec-web]: https://github.com/robbiet480/cec-web
[chbmuc]: http://github.com/chbmuc
[cec.go]: https://github.com/chbmuc/cec
[jasmas-post]: http://www.roomieremote.com/forums/topic/chromecast-hdmi-cec-to-power-on-samsung-tv/#post-16109
[jasmas]: http://github.com/jasmas
[wphdmi]: https://en.wikipedia.org/wiki/HDMI#CEC
[jasmascecd]: http://github.com/jasmas/cecd
[robbiescecd]: http://github.com/robbiet480/cecd
[socat]: http://www.dest-unreach.org/socat/
[libcec]: https://github.com/Pulse-Eight/libcec
[adammw]: http://github.com/adammw
[node-cec]: https://github.com/adammw/node-cec
[usb-cec]: https://www.pulse-eight.com/p/104/usb-hdmi-cec-adapter
[golang]: http://golang.org
[go-pi]: http://dave.cheney.net/2012/09/25/installing-go-on-the-raspberry-pi
[libcec.go]: https://github.com/chbmuc/cec/blob/master/libcec.go#L70
[cectypes.h]: https://github.com/Pulse-Eight/libcec/blob/9be2206f155d4cff55d816e80d6275844cc33b3b/include/cectypes.h#L430-L435
[cec-go-issue]: https://github.com/chbmuc/cec/issues/1
[cec-o-matic]: http://www.cec-o-matic.com/
[cec-specs]: https://xtreamerdev.googlecode.com/files/CEC_Specs.pdf
[roomie-ddk]: http://www.roomieremote.com/wp-content/uploads/2014/02/RoomieDDK-210.zip
[jsonlint]: http://jsonlint.com