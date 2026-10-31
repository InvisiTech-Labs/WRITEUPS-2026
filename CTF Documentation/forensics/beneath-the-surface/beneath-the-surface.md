# Challenge Solution

## Overview

The challenge provides a Microsoft Word `.docx` document with no obvious flag visible when the document is opened normally.

The goal is to investigate the document itself and recover the hidden flag.

## Step 1: Identify the File

Start by checking the supplied file type.

```bash
file doubledouble.docx
```

The file is a Microsoft Word Open XML document.

Modern `.docx` documents use the Office Open XML (OOXML) format. Internally, a DOCX is composed of multiple files stored inside a ZIP package.

This means we can inspect the internal structure of the document using standard ZIP utilities.

---

## Step 2: List the Contents

Before extracting the document, its internal contents can be listed using:

```bash
unzip -l doubledouble.docx
```

This displays the files and directories contained inside the document.

Typical DOCX components include:

```text
[Content_Types].xml
_rels/
docProps/
word/
```

However, while examining the package, an additional directory can be found:

```text
gfx/
```

This is worth investigating.

---

## Step 3: Extract the DOCX

Create a directory and extract the document:

```bash
mkdir extracted
unzip doubledouble.docx -d extracted
```


Among the files is an XML file inside the `gfx` directory.


```text
extracted/gfx/gfxdata.xml
```

---

## Step 4: Inspect the Suspicious XML File

Display its contents:

```bash
cat extracted/gfx/gfxdata.xml
```

Alternatively:

```bash
less extracted/gfx/gfxdata.xml
```

The XML contains a suspicious encoded-looking value.

The characters used in the payload and its structure suggest that it may be Base64 encoded.

---

## Step 5: Extract the Base64 Payload

Copy the value contained inside the XML element. i.e., the payload value 'ENCODED_VALUE'

```xml
<graphicsData>
    <payload>ENCODED_VALUE</payload>
</graphicsData>
```

---

## Step 6: Decode the Payload
Use the **Magic Operation** on CyberChef to extract the flag. Alternatively try different available data format and other operations at cyberchef: [CyberChef](https://gchq.github.io/)

> Using the Magic operation confirms the payload to be a base64 double encoded value (as the filename and file content suggests) of the flag `flag{inm_double_base64_docx_forensics}`


## Key Takeaway

Files should not always be treated only as they appear to the application that normally opens them.

Modern Microsoft Office documents such as `.docx` use the Office Open XML format and contain multiple files packaged together. Examining the underlying file structure and metadata can reveal information that is not visible when the document is opened normally.