# The Image Knows Too Much: Writeup

|||
|-|-|
|**Category**|OSINT|
|**Difficulty**|Easy|
|**File**|`IMG\_6148.HEIC`|
|**Tools**|ExifTool, Google Maps, Street View, web search, online metadata viewers|

## Overview

A photo carries far more than the pixels you see. Inside the file, the metadata records when the picture was taken, what device took it, and where the photographer was standing.

This challenge is split into three questions that all use the **same image**. If you already downloaded it for an earlier part, there is no need to fetch it again.

**What you need to find:**

1. The exact date and time the photo was taken.
2. The hardware model identifier of the phone that took it.
3. The initials of the political party whose poster hangs under a school sign at the location.

\---

## Background: what is EXIF?

EXIF (Exchangeable Image File Format) is a standard for storing details about a photo inside the file itself. Depending on the phone or software, it can include:

* Capture date and time
* Camera maker and device model
* Exposure settings (aperture, ISO, shutter speed)
* GPS position, altitude and direction
* Image size and orientation
* Software and device-specific tags

Investigators use these fields to work out when and where a picture was taken.

## Setup: ExifTool

ExifTool is a command-line program that reads and writes metadata in images, videos and documents.

Check whether it is installed:

```bash
exiftool -ver
```

If not, install it (Kali / Debian / Ubuntu):

```bash
sudo apt update
sudo apt install libimage-exiftool-perl
```

Go to the folder with the image and dump everything:

`
exiftool IMG\_6148.HEIC
```

The output includes EXIF, GPS, QuickTime and Apple-specific tags.

\---

# Part 1: When was the photo taken?

## Find the capture timestamp

Search the output for `DateTimeOriginal`. It holds the moment the shutter fired:

```text
\[ExifIFD] DateTimeOriginal   : 2026:10:27 16:12:17
```

Related fields help confirm it and give the timezone:

```text
\[ExifIFD] CreateDate          : 2026:10:27 16:12:17
\[ExifIFD] OffsetTimeOriginal  : +03:00
\[ExifIFD] OffsetTimeDigitized : +03:00
\[ExifIFD] SubSecTimeOriginal  : 150
```

Put together, the photo was taken on **27 October 2026 at 16:12:17 (UTC+03:00)**, with a fraction of 0.150 s that the question does not ask for.

## Handy commands

Show every time-related tag, grouped and with tag names:

```bash
exiftool -time:all -a -G1 -s IMG\_6148.HEIC
```

Show only the important ones:

```bash
exiftool -DateTimeOriginal -CreateDate -ModifyDate \\
  -OffsetTimeOriginal -SubSecTimeOriginal IMG\_6148.HEIC
```

> \*\*Watch out:\*\* do not use the file's modified time from your operating system. It changes whenever the file is copied, extracted or edited, so it is not the real capture time.

## Flag (Part 1)

Format: `flag{inm\_YYYY:MM:DD\_HH:MM:SS}`

```text
flag{inm\_2026:10:27\_16:12:17}
```

\---

# Part 2: Which phone took it?

## Read the device tags

```text
\[IFD0] Make         : Apple
\[IFD0] Model        : iPhone 13 Pro Max
\[IFD0] HostComputer : iPhone 13 Pro Max
\[IFD0] Software     : 27.0.1
```

So the maker is Apple and the phone is an **iPhone 13 Pro Max**. But the question asks for the **hardware model identifier**, which is a different thing.

## Three names for one phone

|Type|Example|Meaning|
|-|-|-|
|Marketing name|iPhone 13 Pro Max|The name used in advertising|
|Hardware model identifier|`iPhone14,3`|Apple's internal code for the model|
|Regional model number|`A2484`|A number tied to a specific regional variant|

They are not interchangeable, so make sure you give the one that was asked for.

## Look up the identifier

The EXIF data gives the marketing name but not the internal identifier, so verify it from a reliable outside source. Search for:

```text
iPhone 13 Pro Max hardware model identifier
```

Good references:

* [EveryMac identifier lookup](https://everymac.com/ultimate-mac-lookup/?identify=iPhone14,3)
* [Apple Support: identify your iPhone model](https://support.apple.com/en-us/108044)

Both map the iPhone 13 Pro Max to **`iPhone14,3`**.

## Optional: check for hidden tags

You can list unknown and extra tags to see whether the identifier is embedded anywhere:

```bash
exiftool -a -u -G1 -s IMG\_6148.HEIC
```

There is no guarantee it will appear. If it does not, rely on the external lookup above.

## Flag (Part 2)

Format: `flag{inm\_modelidentifier}`

```text
flag{inm\_iPhone14,3}
```

\---

# Part 3: Which political party?

## The question

Using the same image, work out where it was taken and look closely at the surroundings.

**What are the initials of the party whose campaign poster is directly below the school sign?**

Format: `flag{inm\_party}`

## Step 1: Get the GPS coordinates

```text
\[GPS] GPSLatitude  : 1 deg 16' 0.10" South
\[GPS] GPSLongitude : 36 deg 49' 20.22" East
\[GPS] GPSAltitude  : 1668.471056 m
```

## Step 2: Convert to decimal degrees

Google Maps likes decimal degrees. The formula is:

```text
decimal = degrees + (minutes / 60) + (seconds / 3600)
```

South latitudes are negative and east longitudes are positive. This gives roughly:

```text
Latitude  : -1.266694
Longitude : 36.822283
```

ExifTool can do the conversion for you with `-n`, which asks for raw numeric values:

```bash
exiftool -n -GPSLatitude -GPSLongitude IMG\_6148.HEIC
```

## Step 3: Open it in Google Maps

Paste the coordinates into the search bar:

```text
-1.266694, 36.822283
```

or open the link directly: [https://www.google.com/maps?q=-1.266694,36.822283](https://www.google.com/maps?q=-1.266694,36.822283)

The pin lands in the **Parklands** area of Nairobi, which is your starting point.

## Step 4: Explore the surroundings

1. Find **Limuru Road** near the pin.
2. Drop into **Street View** and move along the road.
3. Look for the **Aga Khan school sign**.
4. Look at what is mounted **directly under** the sign.
5. Read the initials on the campaign poster.

Do not stop once you have identified the school. The goal is the poster underneath it.

> \*\*Tip:\*\* if Street View does not show the exact scene, check the imagery dates, look at nearby street-level views, and compare the road layout and landmarks with the original photo.

## Step 5: Read the poster

The poster shows the initials **PDU**, which stands for the **Party of Democratic Unity**.

## Flag (Part 3)

Format: `flag{inm\_party}`

```text
flag{inm\_PDU}
```

\---

# More tools and options

ExifTool is the main tool, but these can help too.

## Online metadata viewers

Use these if ExifTool is not available.

|Site|How to use it|
|-|-|
|[Metadata2Go](https://www.metadata2go.com/)|Upload the image, run the analysis, look for `DateTimeOriginal`, `Make`, `Model` and the GPS fields|
|[Jimpl](https://jimpl.com/)|Upload the image, review EXIF and GPS, find the timestamp and device model|
|[Exif.tools](https://exif.tools/)|Drag the image in and inspect the metadata sections|

Support for HEIC and Apple-specific tags differs between sites. If a field is missing, compare against ExifTool.

> \*\*Privacy:\*\* avoid uploading sensitive photos to third-party services. For anything sensitive, analyse the file locally.

## More ExifTool commands

GPS information:

```bash
exiftool -gps:all -G1 -s IMG\_6148.HEIC
```

Camera and exposure settings:

```bash
exiftool -Make -Model -LensModel -FocalLength \\
  -FNumber -ExposureTime -ISO IMG\_6148.HEIC
```

File details:

```bash
exiftool -FileName -FileType -FileSize \\
  -ImageWidth -ImageHeight -Orientation IMG\_6148.HEIC
```

Export the metadata:

```bash
exiftool -json IMG\_6148.HEIC > metadata.json
exiftool -csv  IMG\_6148.HEIC > metadata.csv
exiftool -h    IMG\_6148.HEIC > metadata.html
```

Verbose output for tricky files:

```bash
exiftool -v3 IMG\_6148.HEIC
```

\---

# Final answers

|Question|Flag|
|-|-|
|1. Date and time|`flag{inm\_2026:10:27\_16:12:17}`|
|2. Phone hardware identifier|`flag{inm\_iPhone14,3}`|
|3. Party initials|`flag{inm\_PDU}`|

# Key takeaways

* Metadata can reveal facts that the picture itself does not show.
* `DateTimeOriginal` is the best starting point for the capture time, and the timezone offset matters when reading it.
* A phone has a marketing name, a hardware identifier and a regional model number. They are different values.
* GPS coordinates can lead you to real streets, buildings and signs, which supports further OSINT work.
* Not every answer is inside the file. Combine metadata with outside research and careful observation.
* Keep facts that came straight from the image separate from conclusions you confirmed elsewhere.

**The image knows more than it shows. The skill is knowing where to look and how to verify what you find.**

