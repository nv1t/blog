---
title: Reverse Engineering and Dismantling Kekz Headphones
date: 2024-10-02
tags:
- 'reverse engineering'
- 'children audio device'
published: true
slug: kekz-headphones
layout: post
---

Close to a year ago, I stumbled upon the Kekz Headphones, which seemed like an interesting approach on the whole digital audio device space. They claimed to work without any internet connection and all of the content already on the headphones itself.
I was intrigued, because there were some speculations going around, how they operate with those "Kekz"-Chips.

I invite you to join me on a journey into the inner workings of those headphones. We will talk about accessing the encrypted files on the device, breaking the crypto and discovering disclosure of data from customers.

<!--more-->

# Opening the headphones
I sourced my headphones from "Kleinanzeigen" (something like craigslist, or facebook marketplace) to keep the research costs low and maybe get some cookies with it. I got the wonderful colour red.
![Headphones opened up, PCBs hanging out, Kekz chips are lying to the left.](/img/2024/9fedcf0c1ac98650a8055d6744523e91.png)
After opening up the headphones, you will have 2 PCBs which are connected by 7 wires. Two speakers and a battery. The chinese lettering in the silk layer is just the colour description of the wires itself. You don't see any interesting breakout for anything here. The Pin-Row in the middle is for the NFC antenna on the other side of the board. You see two Vias with the label `DP` and `DM`, which is on the USB line. This will be interesting at a later stage.

![PCB of one of the ears, which has the important chips on it. Description is in the text.](/img/2024/9c2962c7adb42ddbf4c88bb9f44b7d7a.jpg)
The first thing that stands out is a Jieli Chip, which appears to be the core component of the entire headset. These chips are mostly used in cheap Bluetooth hardware and are difficult to determine which version is currently running. From a quick search i think this Chip (`AC21BP0H733-51C8`) is probably some version of the `AC6951C`.

The chip on the bottom, `TSC9883`, is a NFC Reader IC, which I don't care for.

It has two infrared proximity sensors to detect the ear and cookie insertion to prevent from reading a cookie more than once and stop when taking off the headphones.

On the right of the PCB you see an SD cardholder, which has a 32gb SD Card on the inside. The SD Card has a Fat32 Filesystem with 276 directories. There is an update, which ups that to around 369 directories. Each directory has multiple files with the extension `kez`, which are most likely encrypted.

![Directory and Filelisting from the SD Card](/img/2024/87e3e2c1ef721f3e60733fc2e0e3149e.png)

Interestingly, I was looking at the SD Card before and I connected the headphones to the accompanied Windows Application. After the connection, the files were gone and I was kinda puzzled, until I found the following code in the application:

```csharp
public static void HideFolders()
{
	if (Globals.Drive == null)
	{
		throw new Exception("Drive not set");
	}
	string drive = Globals.Drive;
	string[] directories = Directory.GetDirectories(drive);
	for (int i = 0; i < directories.Length; i++)
	{
		DirectoryInfo directoryInfo = new DirectoryInfo(directories[i]);
		if (int.TryParse(directoryInfo.Name, out var _))
		{
			directoryInfo.Attributes |= FileAttributes.Hidden;
		}
	}
	new FileInfo(Path.Combine(drive, "kekzId.json")).Attributes |= FileAttributes.Hidden;
}
```

They seem to set the hidden Attribute on the first connect, so the files are not easily discovered.

We have multiple options to attack this system. We can either try to dump the firmware of the main controller chip and reverse engineer that, or we understand, how a "Kekz" works.

As we currently have little information about the controller, and we haven't looked into the NFC Communication yet, let's first check the cookies themselves and go the easy route.

# You get a cookie and you get a cookie.

I have a [Proxmark3](https://proxmark.com/) lying around and if we try to "just read" a Kekz, it will result in an error, as it seems to be locked:

```
[usb] pm3 --> hf mfu info

[=] --- Tag Information --------------------------
[+]       TYPE: NTAG 213 144bytes (NT2H1311G0DU)

[=] --- Tag Counter
[=]        [02]: 00 00 00
[+]             - 00 tearing ( fail )

[=] --- Tag Signature
[=]     Elliptic curve parameters: NID_secp128r1
[=]              TAG IC Signature: 0000000000000000000000000000000000000000000000000000000000000000
[+]        Signature verification ( fail )

[=] --- Tag Version
[=]        Raw bytes: 00 53 04 02 01 00 0F 03
[=]        Vendor ID: 53, Shanghai Feiju Microelectronics Co. Ltd. China
[=]     Product type: NTAG
[=]  Product subtype: 02, 50pF
[=]    Major version: 01
[=]    Minor version: 00
[=]             Size: 0F, (256 <-> 128 bytes)
[=]    Protocol type: 03, ISO14443-3 Compliant
[?] Hint: try `hf mfu pwdgen -r` to get see known pwd gen algo suggestions
[=] ------------------------ Fingerprint -----------------------
[=] Reading tag memory...
[#] Cmd Error: 00
[#] Read block 0 error
[!] ⚠️  Failed reading card
[=] ------------------------------------------------------------

[=] Tag appears to be locked, try using a key to get more info
```

We could either Brute-Force (not sure, if this will result in a locked chip), or we can just sniff the communication between the headset and the cookie. By holding the proxmark near the reader of the headset and inserting a cookie, we will get the whole communication between the reader itself and this cookie, which reveals the authentication Key.

![Trace of the NFC communication with the password for the chips highlighted in a red box](/img/2024/a1d514983cf67277c9e02790df905202.png)
It then tries to read the 0 block and the 7th block. The 0 block is only the ID of the tag which bears no relevance. If we look into the 7th block and further, though, we can see a string "en002071696263", if we dump the whole cookie. That is quite interesting.

```
[usb] pm3 --> hf mfu dump -k FFFFFFFF
[+] TYPE: NTAG 213 144bytes (NT2H1311G0DU)
[+] Reading tag memory...

[=] MFU dump file information
[=] -------------------------------------------------------------
[...redacted...]
[=] -------------------------------------------------------------
[=] block#   | data        |lck| ascii
[=] ---------+-------------+---+------
[=]   0/0x00 | 53 BA 20 41 |   | S. A
[=]   1/0x01 | 46 70 00 01 |   | Fp..
[=]   2/0x02 | 37 48 00 00 |   | 7H..
[=]   3/0x03 | E1 10 12 00 | 0 | ....
[=]   4/0x04 | 01 03 A0 0C | 0 | ....
[=]   5/0x05 | 34 03 00 FE | 0 | 4...
[=]   6/0x06 | 00 00 00 00 | 0 | ....
[=]   7/0x07 | 65 6E 30 30 | 0 | en00
[=]   8/0x08 | 32 30 37 31 | 0 | 2071
[=]   9/0x09 | 36 39 36 32 | 0 | 6962
[=]  10/0x0A | 36 33 FE 00 | 0 | 63..
[...redacted...]
[=] ---------------------------------
[=] Using UID as filename
[+] saved 236 bytes to binary file /.proxmark3/files/hf-mfu-53BA2046700001-dump.bin
[+] saved 59 blocks to text file ./.proxmark3/files/hf-mfu-53BA2046700001-dump.eml
[+] saved to json file /.proxmark3/files/hf-mfu-53BA2046700001-dump.json
```

We can check our theory by copying over this string to another cookie, and you will discover, it works. Therefore, **we can clone cookies now.**

![Image of a bunny and cat looking alike next to each other.](/img/2024/8ca30bafd2e46943f989143f33dab0b8.png)

Even if the outside looks different, it plays the same content and is a bunny by heart.

At this point, it is possible to read this string and build a database to decrypt all content. We just have access to the ones we have already seen.
As those tags are 13.35Mhz, is is also possible to write them by your Phones NFC (you will see this later on)

# What does this string mean?

We have this weird string `en002071696263` which has something to do with playing the content and we know the content is probably one of those directories we did see earlier on the SD-card.

If we begin to delete one directory after the other, we can determine which directory has the desired content inside. For this cookie, the directory is `0020`. If we look into other cookies we will see the structure is:

| Cookie           | String           |
| ---------------- | ---------------- |
| Cookie Crew 1    | en 0020 71696263 |
| Cookie Crew 1    | en 0020 71696263 |
| Feuerwehrman Sam | en 0002 6161777a |
| Was ist Was      | en 0031 67766172 |
| Hotzenplotz      | en 0006 73657463 |

I can move those files to a directory `4444`, the files will be played, but garbage output. This means the `0020` is important for the decryption phase.
Renaming `0020` to something else will result in an "unbaked" Chip.
If i move the files to a directory `1020`, they will be played just fine, after rebranding the chip to `en102071696263`

We could try more stuff to understand the encryption, but...let us recap

1. the four integers number after `en` is the directory
2. they are partially important for the decryption.
3. four bytes in the end, we don't know the purpose, but they are essential for decryption

# Moar...give me moaaaaar
The only crux is, we only can decrypt stuff we have already seen, but i want to have an attack on everything.

We could brute force the 4 Bytes. Without any further assumption, this would be `255**4` possibilities, which is way to many.

But if we look into the last four bytes more closely, we can assume one last thing:
In our examples, the four bytes are hex representation for four small letters from the alphabet.

With this assumption, we can bring this down to `26**4`. That sounds more reasonable, but can we attack the crypto further?

Lucky for us, they published an application which can write a cookie named "Wunderkekz". This App can encrypt arbitrary MP3 files to the correct `kez`-Fileformat. And more Lucky for us: it is written in csharp.

Therefore, we can take a look into the encryption routine (i translated it to python, variable naming directly from the original decompilation):
```python
str_crumb_hex = sys.argv[3] # #"E9-F5-33-6B" # Assuming this value based on your previous examples
directory_raw = sys.argv[1]
filename_raw = sys.argv[2]

directory = bytearray(directory_raw, 'ascii')
filename = bytearray(filename_raw, 'ascii')
array = str_crumb_hex.split('-')
b, b2, b3, b4 = [int(value, 16) for value in array]
str_crumb_hex_unpacked = bytearray([b, b2, b3, b4])
b5 = (str_crumb_hex_unpacked[0] ^ directory[0]) >> 4
b6 = (str_crumb_hex_unpacked[1] ^ directory[1]) >> 5
b7 = (str_crumb_hex_unpacked[2] ^ directory[2]) >> 3
b8 = (str_crumb_hex_unpacked[3] ^ directory[3]) >> 2

array3 = bytearray([b5, b6, b7, b8])

b9 = (filename[0] + filename[1] + filename[2] + filename[3]) % 10 - 1
if b9 >= 9 or b9 < 0:
    b9 = 6

with open("%s/%s.mp3" % (directory_raw,filename), 'rb') as file_stream:
	array4 = bytearray(file_stream.read())
	array5 = bytearray(len(array4))
	for i in range(len(array4)):
		array4[i] ^= array3[i % 4]
		array5[i] = ((array4[i] >> (8 - b9)) | (array4[i] << b9)) & 0xFF

with open("%s/%s.kez" % (directory_raw,filename),'wb') as fh:
	fh.write(array5)
```

- Opens the MP3 file for reading.
- Reads the entire file into a byte array `array4`.
- Creates a new byte array `array5` to hold the encrypted data.
- Iterates through `array4`, performing the following operations on each byte:
    - XORs the byte with an element of `array3` (which is some kind of key) (selected in a round-robin fashion).
    - Performs a bitwise rotation on the byte, using `b9` as the shift count, and stores the result in `array5`.
- Closes and disposes of the input file stream.

# Attacking the Crypto

As the shift and 4 Byte XOR key depends on the directory name and unknown 4 Byte Hex Key, we can pre calc a brute-force table which significantly limits the keyspace. We can safely assume the unknown 4-Byte Hex Key to be in a printable state.

This can be seen in the source code of the Kekz App As well (Source: `public static bool GenerateKekzCryptFiles`)

```csharp
string s = RandomGenerator.RandomString(4, lowerCase: true);
strCrumbHex = BitConverter.ToString(Encoding.ASCII.GetBytes(s));
```

The following factors reduce the keyspace significantly:
1. **shifts reduces the keyspace down to 18 bits**:
	- **`b5`** has 16 possible values (because it’s reduced to 4 bits after the shift).
	- **`b6`** has 8 possible values (because it’s reduced to 3 bits after the shift).
	- **`b7`** has 32 possible values (because it’s reduced to 5 bits after the shift).
	- **`b8`** has 64 possible values (because it’s reduced to 6 bits after the shift).
2. **XOR with the directory**: Each character in the `directory` string has a limited value range from 48 (ASCII for '0') to 57 (ASCII for '9'), or just 10 different possible values for each character.
3. **Collision**: After the XOR and bit shifts, many different input values may end up producing the same result. The bit shifts cause significant information loss, and many different values could end up mapping to the same `b5, b6, b7, b8` combination, leading to a large number of collisions. These collisions reduce the number of unique keys generated.

By writing all possible keys into a Dictionary, we don't need to sort and uniq an array afterward.
This results in ~56 possible keys to decrypt the content.

```python
characters = "abcdefghijklmnopqrstuvwxyz"

def keygen(l):
	yield from itertools.product(*([l] * 4))

def pre_calc_array3(directory, filename):
	ret = {}
	for x in tqdm(keygen(characters),total=len(characters)**4):
		str_crumb_hex = '-'.join([hex(ord(i))[2:] for i in x])
		array = str_crumb_hex.split('-')
		b, b2, b3, b4 = [int(value, 16) for value in array]
		str_crumb_hex_unpacked = bytearray([b, b2, b3, b4])
		b5 = (str_crumb_hex_unpacked[0] ^ directory[0]) >> 4
		b6 = (str_crumb_hex_unpacked[1] ^ directory[1]) >> 5
		b7 = (str_crumb_hex_unpacked[2] ^ directory[2]) >> 3
		b8 = (str_crumb_hex_unpacked[3] ^ directory[3]) >> 2
		array3 = bytearray([b5, b6, b7, b8])

		ret[f"{array3[0]},{array3[1]},{array3[2]},{array3[3]}"] = array3
	return ret
```


Currently, i take one file, calculate the shift `b9` and decrypt the file multiple times for every possible key found from the `pre_calc_array3` method

```python
print("Starting Brute Force")
for i in tqdm(array3_poss):
	na = i.replace(',','-')
	array3 = array3_poss[i]
#return True
	with open("%s/%s/%s.kez" % (location,directory.decode('utf-8'),filename.decode('utf-8')),'rb') as fh:
		array6 = bytearray(fh.read())
		array4_reversed = bytearray(len(array6))
		for i in range(len(array6)):
			# Reverse the bitwise rotation
			array4_reversed[i] = ((array6[i] << (8 - b9)) | (array6[i] >> b9)) & 0xFF
			# Reverse the XOR operation
			array4_reversed[i] ^= array3[i % 4]


	with tempfile.NamedTemporaryFile('wb') as fh:
		fh.write(array4_reversed)

		returned_output = os.popen("mpck -q %s" % (fh.name)).read()
		if ": Ok" in returned_output:
			break
else:
	return False
```

The Main Problem relies on checking for a valid MP3 files. Because of the shift and the 4 byte xor key you need to check every MP3 Frame, which takes time on larger files.
I currently use an external tool called "[checkmate](https://github.com/Sjord/checkmate)". It has the most robust MP3 validity solution. It basically checks every Frame. (Maybe Fork and implement it in Python? )

## Do you have an App for that?
I've created, for my use and not publication, a small application to read and write the Cookies with my mobile phone. "Kekzmonster" takes in QR Codes, or reads the cookie with NFC and can back up all cookies in my possession. It is not intended to unlock content, which i don't own.

![Screenshot of my Application Kekzmonster.](/img/2024/749b63fc6e776f7ce19aad16eca2d7dc.png)

The Encryption/Decryption String can be written to any cookie with this application as well.

# Files without breaking the headphones
We can now encrypt, decrypt and brute force the cookie content, but you won't get ny files onto or from the headset on your own, without opening up the headphones and accessing the SD Card. Connecting the headphones to an USB port only charges them and they are listed within Linux as HID Device:
```
Bus 003 Device 012: ID 33f5:0001 Kekz Gmbh kekz headphone
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               1.10
  bDeviceClass            0
  bDeviceSubClass         0
  bDeviceProtocol         0
  bMaxPacketSize0        64
  idVendor           0x33f5
  idProduct          0x0001
  bcdDevice            1.00
  iManufacturer           1 Kekz Gmbh
  iProduct                2 kekz headphone
  iSerial                 3 2021082200001002
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength       0x0022
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0
    bmAttributes         0x80
      (Bus Powered)
    MaxPower              100mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           1
      bInterfaceClass         3 Human Interface Device
      bInterfaceSubClass      0
      bInterfaceProtocol      0
      iInterface              0
        HID Device Descriptor:
          bLength                 9
          bDescriptorType        33
          bcdHID               2.01
          bCountryCode            0 Not supported
          bNumDescriptors         1
          bDescriptorType        34 Report
          wDescriptorLength      27
         Report Descriptors:
           ** UNAVAILABLE **
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x82  EP 2 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0008  1x 8 bytes
        bInterval               1
Device Status:     0x0000
  (Bus Powered)
```

We talked about the two vias `DP` and `DM`. Interestingly, we are not finding any other connection to the chip, no UART, no JTAG, nothing.
Looking into Jie-li, they are pretty weird chips. The documentation and various sources, state they are programmed over the normal USB Data lines. There are two main methods to control them.

## Signaling DP/DM on USB
The normal way to put the chip into DFU mode would be sending a custom pullup/pulldown over the `D+` and `D-`:
![](/img/2024/ad221d82128b7872b20afc5731412ff7.png)
After this signal, `D+` and `D-` gets pulled to Ground for `2ms` and the device boots up into the DFU Mode with uboot.

There are special programmer to achieve this, but i've seen a post, where somebody build his/her own programmer with an arduino.

I tried to achieve this with a raspberry pi pico and couldn't get the Chip into DFU mode. I even tried to use the special programmer for this, but still...no luck.

In addition, i thought, the windows application does some magic to connect them and read/write content of the headphones. How?! That has to work without any extra hardware and you don't have such control over the USB data lines from an operating system application.

## HID Communication
The other more convenient option for these chips are: they might have special commands over HID which reconnect them in different stages. These are not really documented and can be different for each chip, as it depends on the firmware (i think so, that is what i got in rough translations).

### DFU Mode
Using the Python HID Library, this is effortless. The important thing to know is the `dfu_payload` which get's send to the device.
```python
dfu_payload = [0, 85, 170, 1, 2, 3, 4, 170, 85]

device = hid.device()

device.open(vendor_id,product_id)
print(f"HID: Found Device: {device.get_manufacturer_string()} {device.get_product_string()}")

data = bytearray(byte_array_left_pad([0, 33, 9, 0, 2, 1, 0, 64, 0], 0, 65))
buffer_payload = bytearray(byte_array_left_pad(dfu_payload, 0, 65))

device.write(data)

try:
	device.write(buffer_payload)
except Exception:
	print("\nCommunication Error or DFU Success...")
```

You can now flash new firmware on the chip.

### Connecting to copy
The same will go for the connection of the headphones as "normal USB Stick". The payload is a little bit different:

```python
connect_payload = [0, 85, 170, 3, 1, 41, 40, 170, 85]
```

After sending this payload, the headphones disconnect and reconnect as a normal USB Stick.
Keep in mind, that if you run or ran the application, the files are with the Windows hidden attribute.

Success...we can now encrypt custom files, put them into custom directories and write our own cookies.

**We now fully pwn the headphones.**

# Other Discoveries
Browsing through the source code of the application and website, i found other things, which are weren't mentioned before.

## Creating a list of all public cookies
There are multiple cookies, which are not public yet, but as we know from above, we can decrypt all of them. Unfortunately, we have no idea, what is in those directories.
But, i found a pretty neat trick to generate a list of all already public cookies:

The Kekz Webshop is/was built upon a WordPress installation. Fortunately for us, the upload's directory has directory listing enabled. I can download all images ever shown in this webshop.
![Directory Listing of the 2024/11 uploads directory of the Kekz store](/img/2024/35e598415e4725d4c97b4ce7a5e14b80.png)

These files are pretty important for us, because you see, every cookie has an ID, for "Raeuber Hotzenplotz", it is 1-1.0066 and interestingly, the directory is "0006".
![close Up Picture of a cookie with the ID highlighted with a red box](/img/2024/b45c09c7774d501681d013c89f0dd13c.png)
If we look into all the images, we can see product images from the packaging as well. This packaging has a barcode with this ID as well.
![Partial Image of the back packaging of a cookie.](/img/2024/8f2f2a131f5e45e8890c7a60319f7cb8.png)
Therefore, we can automate downloading all images and scanning for barcodes and extracting the directory.

**With this information, we can determine about 1/3 of the content is officially released already in the shop.**

## Moar Wunderkekze
Apparently the directories 0990 until 0996 are used for the WunderKekzChips, whereas Green, Orange and Purple are already in circulation.

Maybe the plan is to add more cookies to the mix.

```csharp
return wChip switch
{
	WunderkekzChipEnum.Green => "0996",
	WunderkekzChipEnum.Orange => "0995",
	WunderkekzChipEnum.Purple => "0994",
	WunderkekzChipEnum.NineThree => "0993",
	WunderkekzChipEnum.NineTwo => "0992",
	WunderkekzChipEnum.NineOne => "0991",
	WunderkekzChipEnum.NineZero => "0990",
	_ => "0035",
};
```

## User Data collection
This topic is not so nice.

While looking through the application, i discovered some not so nice stuff, which wasn't mentioned in the privacy policy anywhere ([Archive](https://web.archive.org/web/20240305082206/https://store.kekz.com/datenschutzerklaerung/)). Point 1.10 about the Kekz App was added at a later stage, after my disclosure emails.

### ID3 Tags
ID3 tags are metadata containers used to store information about an MP3 audio file, such as the song's title, artist, album, and other details. They help media players and libraries organize and display information about the audio files. The ID3 tags are stored within the MP3 file itself.
If you are using an Wunderkekz from the Kekz company, and you use the standard windows application (because there is no other), the ID3 tags are uploaded to an Azure Cosmos database.

```csharp
WunderkekzUploadMetadata wunderkekzUploadMetadata = new WunderkekzUploadMetadata();
try
{
	FileInfo fileInfo = new FileInfo(path);
	wunderkekzUploadMetadata.FileName = fileInfo.Name;
	wunderkekzUploadMetadata.FileSize = fileInfo.Length;
	wunderkekzUploadMetadata.EventId = Globals.CurrentEventId;
	using TagLib.File file = TagLib.File.Create(path);
	wunderkekzUploadMetadata.Id3Title = file.Tag.Title;
	wunderkekzUploadMetadata.Id3Artist = file.Tag.FirstPerformer;
	wunderkekzUploadMetadata.Id3Album = file.Tag.Album;
	wunderkekzUploadMetadata.Id3Year = (int)file.Tag.Year;
	wunderkekzUploadMetadata.Id3Track = (int)file.Tag.Track;
	wunderkekzUploadMetadata.Id3Genre = file.Tag.FirstGenre;
	wunderkekzUploadMetadata.Id3Comment = file.Tag.Comment;
	return wunderkekzUploadMetadata;
}
catch (Exception ex)
{
	Trace.WriteLine(ex.Message);
	return wunderkekzUploadMetadata;
}
```

### Geolocation
Furthermore, the application tries not only uploading the ID3 Tags, but also geolocation data, which is most likely gathered from Wi-Fi triangulation from windows itself.

The MainView calls a GeoLocation Service:
```java
public async Task<string> GetCurrentLocation()
{
	string strLocation = null;
	try
	{
		Location lastLocation = await Geolocation.Default.GetLastKnownLocationAsync();
		Location location = (await Geolocation.Default.GetLocationAsync()) ?? lastLocation;
		if (location != null)
		{
			DefaultInterpolatedStringHandler defaultInterpolatedStringHandler = new DefaultInterpolatedStringHandler(1, 2);
			defaultInterpolatedStringHandler.AppendFormatted(location.Latitude);
			defaultInterpolatedStringHandler.AppendLiteral(":");
			defaultInterpolatedStringHandler.AppendFormatted(location.Longitude);
			strLocation = (Globals.GeoData = defaultInterpolatedStringHandler.ToStringAndClear());
		}
		return strLocation;
	}
	catch (Exception ex)
	{
		Console.WriteLine(ex.Message);
		return strLocation;
	}
}
```

This is also save within the Cosmos DB as seen in already present locations:
```json
{
  "id": "29c01f2e-f4cd-4671-9257-a7432b15fc0d",
  "EventTypeId": "1",
  "DeviceGuid": "3cf4caf5-3f6e-4199-bbf0-ba1abe69a6e9",
  "WunderkekzId": "994",
  "GeoLocation": "48,xxxxxxxxxxxxxx:11,xxxxxxxxxxxxxx",
  "UploadedAt": "2023-10-24T21:03:28.4478194+02:00",
  "_rid": "E9MJAL2dw5WRAAAAAAAAAA==",
  "_self": "dbs/E9MJAA==/colls/E9MJAL2dw5U=/docs/E9MJAL2dw5WRAAAAAAAAAA==/",
  "_etag": "\"3e009e56-0000-0d00-0000-653815010000\"",
  "_attachments": "attachments/",
  "_ts": 1698174209
}
{
  "id": "e3e1482f-1b62-4780-aff1-70593aa79d56",
  "EventTypeId": "1",
  "DeviceGuid": "a7bc4474-570d-4ece-b1f5-aa3e9a36c1ce",
  "WunderkekzId": "995",
  "GeoLocation": "48,xxxxxxxxxxxxxx:11,xxxxxxxxxxxxxx",
  "UploadedAt": "2023-10-24T21:17:59.1191164+02:00",
  "_rid": "E9MJAL2dw5WSAAAAAAAAAA==",
  "_self": "dbs/E9MJAA==/colls/E9MJAL2dw5U=/docs/E9MJAL2dw5WSAAAAAAAAAA==/",
  "_etag": "\"3e009f58-0000-0d00-0000-653818690000\"",
  "_attachments": "attachments/",
  "_ts": 1698175081
}
```

Because the Geolocation data is only cross-referenced with die Device GUID and not the content itself, the enforcement for regional content does not make sense in their newest copy of the privacy policy.

### PII Data
There is some PII Data involved, but i think those are just test data from some ordering processes. Even the headphones can only be cross-linked to the geolocation data and not the content played.
It could maybe be cross-referenced with the time, but haven't checked into that, as the main concern is the data being public.


### Who is Special?
In addition, the connection string to this Azure cosmos database is accessible within the Decompilation of the application itself and therefore is disclosed.

So everybody looking into the source code of the application can get information on usage of the headphones, location of some of the headphones and files listened too.

![Heatmap of all Geolocation Data in Germany from the headphones usage](/img/2024/ddb71ee1385f009813e51273e8dd6d1d.png)
There is some usage in Dublin as well, but only wanted to include the DACH Region.

Normalizing ID3 Tag Meta Data from various sources is really cumbersome. I tried, but i gave up pretty quickly, because it was just my own curiosity what kids are listening to these days. Let me say that: Bibi Blocksberg, Bibi & Tina, Benjaming Bluemchen, Paw Patrol, Drei Fragezeichen, and various songs, are the all time favourits. (for the english speaking community: except Paw Patrol, are all Children Listening experiences from germany, which exist since 1970 or 1980)

# Disclosure
**19.10.2023**: i reached out to the CTO of Kekz, who told me, he developed the headphones, but is no longer associated with the company. To my knowledge, he forwarded the information to the CEOs.
Never heard back.

**27.02.2024**: i reached out to the CEOs of Kekz (Adin and Carl) with my security concerns. Never heard back.
	-> A few weeks later, the privacy policy was changed to include the Kekz-App, therefore i conclude my email was read

# Open Questions
- What is the full functionality of the Jieli-Chip? These chips are strange and challenging to identify. I only guessed which Chip it could be and got lucky with the HID Interface through the Windows Application
- What does the other Jieli-Chip on the other PCB Do?
- A full application to create a custom SD Card with a content manager to not only support the content  already present on the Kekz Headphones, but furthermore all non-taken directories.
- Are there more HID commands?
- How good is the Geolocation Data sourced from a Laptop, which is probably triangulated Wifi Signals? Do the 30m-500m from the privacy policy hold up, or can it be narrowed down?
- What is this PII Data in the Azure cosmos database?

There are most likely more open questions on this one. You are welcome to research further on your own.

# References

- **Kekz Information:**
	- https://futurezone.at/produkte/kekz-kopfhoerer-im-test-kinder-in-ihrer-eigenen-welt/401978339 (first article i have read about the Headphones)
    - https://stadt-bremerhaven.de/kekz-drahtlose-kinderkopfhoerer-nach-dem-tonies-prinzip/ (Kekz article of Caschys blog, with wrong information in the comments on how they operate)
	- https://store.kekz.com/haendlersuche/ (Kekz - Store search)
	- https://apps.microsoft.com/detail/9NXL6Q53G5RX?hl=de-de&gl=DE (Kekz application)
- **Checking MP3s for validity:**
	- https://github.com/Sjord/checkmate (checks every frame, but not python variant exists)
	- https://peterextexia.com/blog/verifying-that-an-mp3-file-is-valid-in-python/ (it kinda works, but get's false positives and because some mp3 decode as valid, this does not work)
- **Jieli-Chip:**
	- https://www.zh-jieli.com/ (chip production)
	- http://www.yunthinker.com/FileUpLoad/DownLoadInfosFile/637729990813300469.pdf (potential Chip)
	- https://github.com/christian-kramer/JieLi-AC690X-Familiarization (Adventures in figuring out how this incredibly ubiquitous, yet incredibly mysterious integrated circuit works.)
	- https://github.com/kagaimiq/jl-uboot-tool/tree/main (chip and protopcol description)
	- https://github.com/kagaimiq/jielie/tree/main (jielie nice!)
	- https://el.jibun.atmarkit.co.jp/thousandiy/2022/09/18_bluetooth_audio_soc.html (Which Chips are built into cheap Bluetooth Speaker)
- **Misc:**
	- https://www.luther-lawfirm.com/newsroom/blog/detail/reverse-engineering-nach-dem-geschaeftsgeheimnisgesetz-geschgehg-vertragliche-ausschlussmoeglichkeiten#:~:text=a%20GeschGehG%20ist%20das%20Reverse,Gegenst%C3%A4nden%20uneingeschr%C3%A4nkt%20erlaubt. (Reverse Engineering Rechtliche Lage)
