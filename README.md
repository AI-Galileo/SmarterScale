# SmarterScale
# Starter Kit
Resources for Developing a BLE Android App for
Etekcity Smart Scales
Reverse Engineering Etekcity BLE Protocols
Building a custom app for the Etekcity ESB4074C (and similar smart scales) first requires understanding
their Bluetooth Low Energy (BLE) protocol. Several community-driven reverse engineering efforts provide a
roadmap:
Capturing BLE Traffic: A common approach is to enable Android’s Bluetooth HCI snoop logging and
use the official VeSync app to record BLE packets. The openScale project’s guide outlines this
process: enable the HCI snoop log in developer options, take multiple weigh-ins (with known weights
or added items), then retrieve the btsnoop_hci.log via ADB for analysis . By noting your
exact weight and other metrics during each trial, you can later search the log for those values in hex
(considering endianness) to identify where the scale transmits the data . Tools like Wireshark help
filter and interpret these logs.
Using BLE Scanner Apps: Tools such as LightBlue (on iOS/Android) or nRF Connect can scan for the
scale and enumerate its GATT services. In one example, an Etekcity “Nutrition Scale” advertised an
obvious name and was interrogated with LightBlue to reveal its services and characteristics . This
confirmed the presence of a custom service expected to hold the weight reading. If the scale doesn’t
broadcast a recognizable name, you may use RSSI or MAC address to identify it . Scanning apps
also allow manually reading/writing characteristics, which is useful for experimentation.
Community Blog Write-ups: A detailed blog post on paulbanks.org demonstrates reverse
engineering an Etekcity ESF24 fitness scale by logging the VeSync app’s BLE communication .
The author retrieved a BLE packet trace and identified a proprietary protocol: the scale exposes a
standard Battery Service (UUID 0x180F for battery level) and a custom service 0xFFF0 with a notify
characteristic 0xFFF1 (for receiving weight data) and a write characteristic 0xFFF2 (for sending
commands) . By enabling notifications on 0xFFF1 and observing data, and issuing writes to
0xFFF2 , they deduced how the scale communicates . Notably, the VeSync app sends some
initial commands (e.g. configuration bytes, possibly to set units or user info, and a timestamp)
before the scale starts streaming measurements . This kind of reverse-engineering log is
invaluable for understanding the handshake and data flow.
Reverse Engineering Guides: The openScale project wiki provides a step-by-step “How to reverse
engineer a Bluetooth 4.x scale” guide . It suggests steps from capturing traffic to discovering
services and characteristics (openScale’s dev mode can scan and list all GATT services of the scale)
, and finally analyzing the protocol by searching for known values (like your weight) in the
captured data. This general guide was used by community members to add support for new scales.
•
1 2
2
•
3
4
•
5 6
7
8
9 10
•
11 2
12
1
BLE GATT Characteristics and Packet Structure
Etekcity smart scales do not use the standard Bluetooth SIG Weight Scale profile; instead they use a custom
GATT profile. Common findings from multiple sources include:
Custom Service & Characteristics: As noted, the scales tend to use a vendor-specific service UUID
0000FFF0-0000-1000-8000-00805F9B34FB . Under this service, 0xFFF1 is a notify
characteristic (indicating the scale will push measurement data here), and 0xFFF2 is a write
characteristic for control commands . Before sending weight data, the mobile app typically
enables notifications on the FFF1 characteristic (to subscribe to weight updates) . The app may
also send certain bytes to FFF2 to configure the scale. For example, one reverse-engineered
session showed the app writing a sequence of bytes to set measurement units and sync time right
after connecting . After this setup, the scale begins streaming measurement notifications.
Measurement Broadcasts: Once activated, the scale will send a series of notification packets on
FFF1 as the user stands on it. These packets update in real-time as the weight stabilizes . In the
ESF24 trace, packets prefixed with < 10 0b 15 ...> were repeatedly sent, changing as weight
changed . When the weight reading finalizes, the scale sends a final data packet that often
includes additional metrics. For instance, the final packet for ESF24 contained the stable weight plus
a bio-impedance value (used for body fat) . Immediately after this, the app issued a write ( >
packet) to command the scale to stop transmitting . This implies your custom app may need to
handle a stream of notifications and know when the final reading is reached (often indicated by a
flag or simply no more change in weight and presence of impedance data).
Standard Services: In addition to the custom weight service, Etekcity scales usually implement basic
standard services. The Battery Service (UUID 0x180F) is one – you can read 0x2A19 to get the
battery level . This is handy to implement in your app so you can display the scale’s battery status.
Also, the Generic Access and Generic Attribute services (0x1800, 0x1801) are typically present by
default on BLE devices (not custom to Etekcity). The scale’s device name (e.g. “QN-Scale1” has been
reported as the BLE name for at least one Etekcity model) is available via the Generic Access service
.
Decoding Weight Data Packets
One of the biggest tasks is decoding the raw byte values from the notify characteristic into human-readable
measurements. Community reverse engineering has uncovered important details:
Payload Format: The Etekcity Luminary kitchen scale (and likely others) sends a 17-byte payload on
its notify characteristic for each measurement update . By comparing multiple readings,
developers identified which bytes correspond to weight. In the Luminary’s case, bytes 11 and 12 (0-
indexed) contain the weight reading in little-endian format . For example, a displayed weight
of 98.0 g appeared in the notification as the hex sequence D4 03 (which is 0x03D4 = 980 in
decimal) . Similarly, 18.0 g was B4 00 (0x00B4 = 180) . The scale was effectively reporting
weight in tenths of grams. Converting the little-endian byte pair to a number and dividing by 10
yielded the weight value .
•
13
14
8
9
•
15
15
16
17
•
13
18 19
•
20 21
22 23
24 22
22
2
Sign and Units: These scales can report both positive and negative values (e.g. taring or calibration
might produce a negative reading). A specific byte often acts as a sign indicator. In the Luminary
example, byte 10 was the sign byte (0x00 for positive, and a non-zero value if negative) .
Additionally, bytes were identified that represent the unit and measurement mode. By pressing the
physical unit button on the scale, one can see certain bytes change in the notification payload . In
Nick Gregorich’s analysis of the kitchen scale, byte 14 indicated the unit (e.g. grams vs ounces) and
byte 15 was a “media” or mode indicator . For a body scale like the ESB4074C, unit bytes might
serve a similar purpose (kg vs lb), or the scale might even encode a user profile ID or measurement
type in those bytes.
Impedance and Body Metrics: Smart body fat scales measure bio-impedance via foot sensors. The
raw impedance value is usually included in the data once a stable weight is obtained. In the ESF24
log, the final data packet had additional bytes beyond the weight; those correspond to impedance/
resistance . Community developers confirmed that one must parse this value to compute body
metrics. For example, a Home Assistant integration for Etekcity scales notes it captures weight and
impedance from the device and then performs body composition calculations in software . The
scale typically does not directly send body fat %, BMI, etc.; instead, your app must calculate these
using standard formulas (inputs: weight, impedance, user height/age/sex). The Etekcity ESF-551
Python library demonstrates deriving metrics like BMI, body fat %, muscle mass, etc., once weight
(kg) and impedance (ohms) are known . If your app only needs weight, you can ignore the
impedance, but for full functionality you’ll want to capture that value and apply a body composition
algorithm (e.g. the built-in formulas used by open-source projects).
Endianness and Scaling: Be mindful of endianness when decoding multi-byte values. As noted,
weight often comes in little-endian. The openScale wiki specifically advises searching for weight in
little-endian byte order in the BLE logs . Also, determine the scaling factor: many scales use one
or two decimal places. For instance, some send weight in hectograms or decigrams. In Nick’s
example, 98.0 g was encoded as 980 (implying one decimal place) . A bathroom scale sending
kilograms might encode 70.3 kg as 703 (if one decimal place) or 7030 (if two decimal places for
70.30). Reverse engineering logs or trial-and-error with known weights can reveal the factor.
CRC/Checksum Bytes: Etekcity’s data packets often have extra bytes that could be reserved or used
for checksums. In the Luminary packet, there were 3 sections of changing bytes, and one might be a
checksum or sequence number . While decoding weight, you might initially ignore bytes that
don’t match known values, but keep in mind they could serve error-checking or other functions. In
practice, many implementations have found that focusing on the weight and impedance bytes is
sufficient to get reliable readings.
Android BLE Development Tips for Smart Scales
With an understanding of the protocol, you can implement the BLE communication on Android. Key
considerations and resources:
Scanning and Identifying the Scale: Use Android’s BLE APIs (or a library) to scan for devices. If the
scale is advertising while active, you can filter by device name or MAC. Some Etekcity scales
broadcast a name like “QN-Scale” or similar, but others might use random addresses. One developer
chose to identify a BLE scale by the first 3 bytes of its MAC address as a unique prefix . In
•
23
25
25
•
26
27
28 29
•
2
22
•
30 31
•
32 33
3
practice, you might simply display a list of BLE devices and let the user pick their scale (by name).
Ensure you have location permissions on Android 11+ to receive BLE scan results (as BLE scans are
tied to location permission due to beacon privacy).
Connecting and Service Discovery: After selecting the device, connect and discover services.
Android’s BluetoothGatt APIs will give you a list of services and characteristics. You should
programmatically look for the known UUIDs (0x180F for battery, 0xFFF0 for the custom scale service,
etc). Once found, enable notifications on the FFF1 characteristic. This involves writing a descriptor
(UUID 0x2902) with the value 0x01 to turn on notifications. Many BLE tutorials cover this process. (If
new to Android BLE, see Google’s BLE guide or sample projects; although not scale-specific, the flow of
scanning, connecting, discovering, and enabling notifications is standard.)
Handling Notifications: When notifications are enabled, the scale will start sending data to your
BluetoothGattCallback.onCharacteristicChanged() handler. Here you’ll receive a byte
array for each update. You’ll need to parse this byte array according to the protocol determined
earlier. For example, you might do:
ByteBuffer.wrap(value).order(ByteOrder.LITTLE_ENDIAN).getShort(10) (if bytes 10-11
contain weight, for instance). A Stack Overflow Q&A shows a scenario of an Android developer
reading a byte array from a BLE scale but needing to convert it to a human weight value . The
solution was to extract the correct bytes (in the correct order) and apply the proper scaling factor.
Ensure you test this parsing with known weight values to verify correctness.
Writing to the Scale (Commands): In many cases, you can get basic weight data just by enabling
notifications. However, to mimic full app functionality, you may implement a few writes. Common
commands include setting the unit or syncing time/user data. For example, one open-source script
demonstrated writing a specific hex string to the scale’s write characteristic to switch units between
kg and lbs . Another example from the ESF24 trace is the app sending the current timestamp
(likely to allow the scale to timestamp its stored measurements) . For a minimal Android app,
sending these is optional – the scale will usually default to some unit and still send weight. But if you
notice the scale isn’t sending impedance or stops transmitting quickly, it might be waiting for a
“configuration” command. Community integrations have revealed some of these command
sequences (which you can hardcode or derive from logs). The p-doyle/Python-ReadBluetoothScale
project on GitHub, for instance, includes commented code and gatttool commands used for Etekcity
scales . Those can guide you if unit switching or other config is needed.
Libraries and Language Choices: While you’re focusing on Android/Java/Kotlin, it’s useful to study
implementations in other languages too. Python is popular for BLE prototyping due to libraries like
bluepy or Bleak . The Bleak library was used by one developer to write a small client that
connects to a Crénot (Etekcity-like) scale and reads data . That code (though in Python) can
clarify the sequence of GATT operations needed. Similarly, the p-doyle script mentioned uses BluePy
to passively listen for the scale’s advertisement and then connect for the reading – an
approach you might mirror on Android by auto-connecting when the device is in range and active.
Android BLE Example Projects: For more concrete examples, you might look at open-source
Android apps that interface with BLE scales. The openScale app (open source on GitHub) supports
many BLE scales – reviewing its code for one of the supported models (e.g. Xiaomi Mi Scale or
others) can show how it structures BLE communication. Another example is a project called “journey-
•
•
34 35
•
36
10
36
•
37 38
18 39
•
4
android-bluetooth-scale” which integrates a Hiweigh scale; while not Etekcity, the code could offer
a template for managing BLE GATT in an Android activity/service (scanning, connecting, notifications,
etc.). Additionally, tutorials like “Real-time Weight data reading from a Bluetooth scale in Flutter”
illustrate cross-platform approaches (Flutter in that case) – the concepts carry over to native
Android.
Open-Source Projects and Code References
Leveraging existing projects can accelerate development. Here are some relevant repositories and
integrations dealing with Etekcity or similar BLE scales:
openScale (Android app): openScale is a popular open-source weight tracking app that works
with many BLE scales. As of now, it doesn’t natively support Etekcity models, but its GitHub issues
and wiki show community interest in adding them. For example, a user requested Etekcity scale
support and the maintainer advised attempting reverse engineering with the provided guidelines
. If you explore openScale’s code, you’ll find how it handles BLE scanning and parsing for other
scales (which often have similarities). openScale’s approach is to abstract each scale’s protocol, so
adding Etekcity would involve writing a new parser for its byte format. Even if you build your own
app, these parsing functions can be a helpful reference for decoding measurements.
Home Assistant Integration: There is a custom Home Assistant integration for Etekcity BLE scales
. This project, available on GitHub (ronnnnnnnnnnnnn/etekcity_fitness_scale_ble), directly
communicates with Etekcity scales from a Raspberry Pi or similar device running Home Assistant. It
automatically discovers scales and pulls weight and impedance in real-time . Notably, this
integration is offline (no VeSync cloud needed) and uses a small Python library called
etekcity_esf551_ble under the hood . Studying that library’s code is highly recommended –
it implements the BLE protocol for the Etekcity ESF-551 model. The README confirms it was only
tested on ESF-551 but might work with other models in the range . The library likely contains the
UUIDs, notification handling, and byte parsing logic in a clear form. By examining it, you can confirm
details like which bytes correspond to impedance, how unit conversion is handled, etc., and mirror
that in your Android code.
Python Scripts and Libraries: Aside from the official integration above, several standalone Python
projects target Etekcity scales:
The “etekcity_scale” library by Paul Banks (banksy-git on GitHub) was a test implementation for the
ESF24 scale . It proved successful in reliably retrieving measurements . Although the code
itself is in Python, it illustrates the sequence of enabling notifications and reading values. Paul’s
accompanying blog and code show how the final output included lines like “19.7 1 434
363” (which likely correspond to 19.7 BMI, 1 maybe a user index, 434 ohm, 363 something) – an
insight into how data might be formatted.
The p-doyle/Python-ReadBluetoothScale script (linked in an OpenMQTTGateway forum post)
focuses on the Etekcity ESF24 and demonstrates a passive listening approach . It waits for
the scale’s advertisement when someone steps on it, then connects to read the weight. This is an
interesting strategy to trigger measurement without constantly polling. The script’s README (and
code comments) provide hints for decoding the BLE advertisements and using gatttool to
40
41
32
• 42
43
•
44
45 27
46
47
•
•
48 49 50
9 16
49
•
18 39
5
interact with the scale . For instance, it provides raw command bytes for switching units
which confirms some of the protocol details.
The hertzg/metekcity project on GitHub is an attempt to reverse engineer the Etekcity Smart
Nutrition Scale (ESN00) . While a food nutrition scale is a different product, it shares BLE
concepts. The developer’s write-up on Dev.to (“Hacking BLE Kitchen Scale”) narrates the process of
finding the right service/characteristic and decoding measurements . In the end, they
identified the same FFF0/FFF1 service on the nutrition scale and managed to read its weight data. If
your focus is primarily the body scale, you might not need details like nutritional info, but this is still
a valuable example of reverse engineering a related Etekcity BLE device.
Personal Projects: Yet more community efforts exist for various BLE scales. For example, a
developer named Stefan Römer documented how he reverse engineered a Crénot GoFit S2 body fat
scale (very similar in concept to Etekcity’s) . He used a Python Bleak client to get the weight value
reliably, though decoding of body fat % was left for future work . His GitHub repo “scalecommunication”
contains the code for connecting and reading that scale . The blog and code
reinforce the idea that once the GATT characteristics are known, fetching the weight is
straightforward – it’s the proprietary impedance and metric calculations that can be tricky. By
reading through such projects, you can gather tips on handling edge cases (e.g. scale not sending
data until a certain sequence is sent, or needing to wake the scale by writing to it).
Home Automation Forums: The desire to integrate Etekcity scales into smart homes has produced
forum threads with useful info. On Home Assistant’s forum, users discussed the lack of official
support in the VeSync (Etekcity) API and turned to direct BLE solutions . Some noted that the scale
shows up in logs as an “Unknown device” with model ID (e.g., ENS-L221S) , which further
confirmed that the cloud API doesn’t handle scales yet. This justified the custom BLE route. On the
OpenMQTTGateway forum, a user even confirmed with Etekcity support that if one leg is amputated,
the scale will never record body metrics – it only sends weight in that case . This kind of anecdote,
while tangential, tells us that the impedance data won’t appear unless the circuit is complete (both
feet on scale), which is something to keep in mind during testing (if you ever see only weight but no
impedance bytes, ensure the person is using it with both feet).
Challenges and Community Workarounds
Developers have encountered a few challenges when working with these scales, along with clever solutions:
Lack of Official Documentation: Etekcity/VeSync do not publish a BLE API for their scales, so
everything is learned through reverse engineering. This means behavior could vary slightly between
models. The community has embraced tools like Wireshark, nRF Connect, and custom scripts to fill
this gap. For example, filtering Wireshark logs by the scale’s MAC and ATT opcodes (0x52 for
notifications, 0x1B for writes) helped isolate the relevant packets . If you run into unknown bytes,
comparing logs from multiple sessions or devices (as openScale suggests) can help differentiate
constant vs variable fields.
Multi-packet Data & Timing: The scales stream multiple packets and possibly expect a confirmation
or stop signal. If your Android app only reads the first notification and disconnects too early, you
might miss the final stable reading or impedance data. The integration and sniff logs show the
51 36
•
52 53
54 55
•
56
57
38
•
58
59
60
•
61
•
6
scale’s final packet contains the complete measurement . It’s wise to keep the connection open
until you’re sure you have the final data (you may detect this by a “stabilized” flag or simply no more
changes for a second or two). Some apps send a specific write (like the 1f05151049 seen in the
log) to gracefully stop the stream , but often simply stepping off will cause the scale to stop on its
own after a timeout. Testing will clarify the needed approach.
Data Interpretation (Beyond Weight): Figuring out the encoding for things like timestamp, user ID,
or advanced metrics can be difficult. In many cases, community projects ignored these at first and
focused on weight. For example, Stefan’s project mentioned he hadn’t cracked the body fat
percentage encoding yet . If you need those, you might incorporate existing formulas or libraries.
The good news is that impedance combined with known user data can compute all metrics –
openScale’s wiki lists formulas under “Body metric estimations” (for BMI, fat%, water%, etc.), and the
Home Assistant integration already implements these calculations in Python. So your app might
retrieve just weight & impedance from BLE and compute the rest in Java/Kotlin using the same
algorithms.
Device Pairing and Security: Most BLE scales use no pairing (just connect and read), but if you
encounter any that require bonding or MITM protection, you’d need to handle BLE pairing in your
app. Etekcity scales used by VeSync don’t appear to require PINs or bonding – the communication is
unencrypted BLE. This is why third-party apps can integrate with them easily. Just ensure your app
gracefully requests Bluetooth enable and permissions from the user. Scales typically advertise only
when active (someone standing on them), so scanning continuously or using a background service
might be needed if you want automatic detection.
Athlete Mode and Other Settings: Some scales have an “athlete mode” or other device-specific
settings (which alter the body fat algorithm). The Home Assistant custom integration explicitly notes
it does not support Athlete Mode yet, and treats all measurements with the standard formula . If
the ESB4074C scale has such modes, you might simply ignore them or allow the user to toggle a
different formula. The BLE protocol likely doesn’t change for athlete mode – it’s probably an app-side
setting for calculation. Just be aware of it as a user expectation (athletes tend to get different body
fat readings).
Reuse of OEM Modules: As observed with the “QN-Scale” identifier, Etekcity might be using a
common BLE scale module that other brands use. QN (QingNiu) is a Chinese manufacturer known
for BLE scale technology. This means that reverse engineering info from other scales (e.g. Renpho,
Yunmai, etc.) can sometimes apply. Check forums or GitHub for projects on those scales – you may
find very similar UUIDs or data formats. In fact, openScale already supports many brands; if one of
them internally matches the QN protocol, you might adapt that parser for Etekcity with minimal
changes.
In summary, the community knowledge base around BLE smart scales is rich, and you won’t be starting
from scratch. By examining the cited GitHub projects, blog posts, and forum discussions, you can piece
together the BLE services and characteristics, the data encoding, and even get sample code for scanning
and reading values. With that foundation, building an Android app to connect to the Etekcity ESB4074C
scale becomes a much more tractable project.
15
17
•
38
•
•
62
•
7
Sources:
Nick Gregorich’s BLE kitchen scale reverse engineering (Etekcity Luminary) – custom service 0xFFF0
and decoding 17-byte weight data .
Paul Banks’s ESF24 scale analysis – GATT details and packet logs showing weight & impedance data
flow .
openScale project wiki – steps to sniff BLE communications and identify characteristic data (general
BLE scale RE guide) .
Home Assistant Etekcity BLE integration – open-source code using a Python BLE library to get realtime
weight and compute body metrics .
Community forums (OpenMQTTGateway, Home Assistant) – user findings on Etekcity scale
advertising name and third-party script to read weight , plus discussions on lack of official
support.
Additional open-source efforts (hertzg’s metekcity for nutrition scales , sroemer’s scalecommunication
for body scale ) which mirror these techniques and can be referenced for
guidance.
How to reverse engineer a Bluetooth 4.x scale · oliexdev/openScale Wiki · GitHub
https://github.com/oliexdev/openScale/wiki/How-to-reverse-engineer-a-Bluetooth-4.x-scale
Liberating BLE data - Nick Gregorich
https://nickgregorich.com/posts/liberating-ble-data/
Sniffing the EtekCity ESF24 Fitness scale -
PaulBanks.Org
https://paulbanks.org/projects/etekcity/
Etekcity ESF24 BLE Weight Scale - Devices not yet supported - Theengs and
OpenMQTTGateway
https://community.openmqttgateway.com/t/etekcity-esf24-ble-weight-scale/1136
GitHub - ronnnnnnnnnnnnn/etekcity_fitness_scale_ble: A Home Assistant custom
integration for Etekcity Bluetooth Low Energy (BLE) fitness scales. Get real-time weight measurements and
body metrics in your smart home setup.
https://github.com/ronnnnnnnnnnnnn/etekcity_fitness_scale_ble
GitHub - ronnnnnnnnnnnnn/etekcity_esf551_ble: This package provides a basic unofficial
interface for interacting with Etekcity ESF-551 Smart Fitness Scale using Bluetooth Low Energy (BLE). It
allows you to easily connect to the scale, receive weight and impedance measurements, and manage the
display unit settings.
https://github.com/ronnnnnnnnnnnnn/etekcity_esf551_ble
Real-time Weight data reading from a Bluetooth Weighing Scale in Flutter -(GAP protocol). | by
Shubham Hande | Medium
https://medium.com/@shubhamhande/real-time-weight-data-reading-from-a-bluetooth-weighing-scale-in-flutter-gapprotocol-
6c9181903915
Newest 'byte' Questions - Page 4 - Stack Overflow
https://stackoverflow.com/questions/tagged/byte?tab=newest&page=4
•
22 23
•
7 15
•
11 2
•
27 63
•
18 39
• 64
37
1 2 11 12
3 4 20 21 22 23 24 25 30 31
5 6 7 8 9 10 13 14 15 16 17 26 48 49 50 61
18 19 39 60
27 44 45 46 62 63
28 29 47
32 33 41
34 35
8
GitHub - p-doyle/Python-ReadBluetoothScale: Passively listen for and read weight from Etekcity
ESF24 Bluetooth scale with python
https://github.com/p-doyle/Python-ReadBluetoothScale
Reverse engineering a BLE body fat scale protocol :: SENVANG IT Solutions
https://senvang.org/posts/2024/09/scale_interface/
journeyapps/journey-android-bluetooth-scale - GitHub
https://github.com/journeyapps/journey-android-bluetooth-scale
Supported scales in openScale - GitHub
https://github.com/oliexdev/openscale/wiki/supported-scales-in-openscale
Add Support for the ETEKCITY Bluetooth Scale · Issue #509 · oliexdev/openScale · GitHub
https://github.com/oliexdev/openScale/issues/509
hertzg/metekcity: ETEKCITY smart nutrition scale protocol ... - GitHub
https://github.com/hertzg/metekcity
Hacking BLE Kitchen Scale - DEV Community
https://dev.to/hertzg/hacking-ble-kitchen-scale-55io
Add support for etekcity smart scale in the vesync integration ENS-L221S-SUS - Third party
integrations - Home Assistant Community
https://community.home-assistant.io/t/add-support-for-etekcity-smart-scale-in-the-vesync-integration-ens-l221s-sus/630181
36 51
37 38 56 57
40
42
43
52 53
54 55 64
58 59
9
