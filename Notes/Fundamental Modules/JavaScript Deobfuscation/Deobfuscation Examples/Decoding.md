# Decoding

When interacting with the application in the previous exercise, the server returned a block of unreadable text rather than a human-readable response:
Decoding

```
willywonka69@htb[/htb]$ curl http://SERVER_IP:PORT/serial.php -X POST -d "param1=sample"

ZG8gdGhlIGV4ZXJjaXNlLCBkb24ndCBjb3B5IGFuZCBwYXN0ZSA7KQo=
```

This behaviour highlights an additional layer commonly used alongside obfuscation. Instead of exposing meaningful strings directly, applications often encode data and decode it only during execution. This approach reduces readability and makes automated inspection more difficult. Obfuscated scripts frequently contain encoded text segments that are transformed back into their original form at runtime.

The following sections cover three encoding techniques that are frequently encountered during code analysis:

base64
hex
rot13

# Base64

Base64 is widely used to represent data using a restricted character set. Encoded output consists only of letters, digits, and the symbols + and /. This makes base64 suitable for transporting data in contexts where special characters may cause issues. Regardless of whether the original input is text or binary, the encoded result always conforms to this limited set.

# Spotting Base64

Base64 strings are usually straightforward to recognise. They contain only alphanumeric characters and often end with one or two = characters. These trailing symbols are padding, which ensures the encoded output length is a multiple of four characters, as required by the base64 specification.

# Base64 Encode

On Linux systems, data can be encoded using base64 by piping input into the base64 utility:
Decoding

```
willywonka69@htb[/htb]$ echo https://www.hackthebox.eu/ | base64

aHR0cHM6Ly93d3cuaGFja3RoZWJveC5ldS8K
```

# Base64 Decode

Decoding base64 data is performed using the -d option:
Decoding

```
willywonka69@htb[/htb]$ echo aHR0cHM6Ly93d3cuaGFja3RoZWJveC5ldS8K | base64 -d

https://www.hackthebox.eu/
```

# Hex

Hexadecimal encoding represents each character using its corresponding value from the ASCII table, expressed in base 16. For instance, the character a maps to 61, b to 62, and c to 63. Reference information for ASCII values can be accessed on Linux systems using the man ascii command.

# Spotting Hex

Hex-encoded data is composed exclusively of sixteen possible characters: the digits 0 through 9 and the letters a through f. This restricted alphabet makes hexadecimal encoding easy to identify during inspection.

# Hex Encode

To convert text into hexadecimal format on Linux, the xxd utility can be used with the -p option:
Decoding

```
willywonka69@htb[/htb]$ echo https://www.hackthebox.eu/ | xxd -p

68747470733a2f2f7777772e6861636b746865626f782e65752f0a
```

# Hex Decode

Reversing hexadecimal encoding can be done using the same tool with the -r option:
Decoding

```
willywonka69@htb[/htb]$ echo 68747470733a2f2f7777772e6861636b746865626f782e65752f0a | xxd -p -r

https://www.hackthebox.eu/
```

# Caesar/Rot13

The Caesar cipher is a classical substitution technique in which letters are shifted forward by a fixed number of positions in the alphabet. One of its most common variants is rot13, which applies a shift of thirteen characters. Applying the same operation twice restores the original text.

# Spotting Caesar/Rot13

Although Caesar-based encodings distort readable text, they often preserve recognisable patterns. For example, protocol prefixes and repeated words may still resemble their original structure, making this encoding identifiable with some familiarity.

# Rot13 Encode

Linux does not provide a dedicated rot13 command, but the transformation can be achieved using character translation:
Decoding

```
willywonka69@htb[/htb]$ echo https://www.hackthebox.eu/ | tr 'A-Za-z' 'N-ZA-Mn-za-m'

uggcf://jjj.unpxgurobk.rh/
```

# Rot13 Decode

Because rot13 is symmetrical, the same command can be used to decode the output:
Decoding

```
willywonka69@htb[/htb]$ echo uggcf://jjj.unpxgurobk.rh/ | tr 'A-Za-z' 'N-ZA-Mn-za-m'

https://www.hackthebox.eu/
```

Rot13 encoding and decoding can also be performed using various online utilities.

# Other Types of Encoding

Beyond these examples, many additional encoding schemes exist. While base64, hexadecimal, and rot13 are among the most frequently encountered, other formats may appear during analysis and require deeper investigation.

When faced with unfamiliar encoded data, the recommended approach is to first identify the encoding method, then locate suitable tools to decode it. Automated identifier tools can assist in this process by analysing patterns and suggesting likely encodings.

In addition to encoding, some obfuscation mechanisms rely on encryption. Unlike encoding, encryption requires a key to restore the original data. If the key is not present within the script, reversing such obfuscation becomes significantly more complex and often requires advanced analysis techniques.
