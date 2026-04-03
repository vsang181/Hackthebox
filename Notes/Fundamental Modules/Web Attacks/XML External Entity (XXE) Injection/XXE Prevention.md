## XXE Prevention

XXE vulnerabilities are easier to prevent than most web vulnerabilities because they rarely stem from poor coding practices by developers. The root cause is almost always outdated XML parsing libraries, which means keeping dependencies current is the single most impactful defence.

***

## Avoiding Outdated Components

Web developers typically do not handle raw XML parsing themselves. Built-in XML libraries do that work, so when a parser is outdated, the entire application becomes exposed regardless of how carefully the rest of the code is written. A practical example is PHP's [libxml_disable_entity_loader](https://www.php.net/manual/en/function.libxml-disable-entity-loader.php), which has been deprecated since PHP 8.0.0 because it allowed developers to enable external entities in unsafe ways, directly enabling XXE.

Modern code editors like [Visual Studio Code](https://code.visualstudio.com/) will flag deprecated functions and suggest alternatives, making it easier to catch these issues before deployment.

Beyond the core XML library, the following components also perform XML parsing and must be kept up to date:

- API libraries such as [SOAP](https://www.w3schools.com/xml/xml_soap.asp)
- SVG image processors
- PDF document processors
- Any third-party packages that touch XML input, managed through tools like [npm](https://www.npmjs.com/)

A full list of vulnerable XML libraries with specific update guidance is maintained in [OWASP's XXE Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html).

***

## Safe XML Configurations

Alongside keeping libraries current, hardening the XML parser configuration adds a second layer of protection. Apply the following settings wherever XML is parsed:

- Disable referencing custom Document Type Definitions (DTDs)
- Disable referencing external XML entities
- Disable parameter entity processing
- Disable support for [XInclude](https://www.w3.org/TR/xinclude/)
- Prevent entity reference loops

These configurations reduce exploitation potential even if a vulnerable library version slips through. That said, they are workarounds rather than fixes. Running a hardened config on top of a vulnerable library is not a substitute for updating the library itself.

***

## Additional Defensive Layers

Error-based XXE relies on runtime errors being visible in the browser. Always implement proper exception handling and disable the display of runtime errors in production web servers.

Where possible, move away from XML entirely. Using [JSON](https://www.json.org/) or [YAML](https://yaml.org/) for data exchange eliminates the XML attack surface by default. This includes replacing XML-based API standards like [SOAP](https://www.w3schools.com/xml/xml_soap.asp) with JSON-based alternatives such as [REST](https://restfulapi.net/).

[Web Application Firewalls (WAFs)](https://owasp.org/www-community/Web_Application_Firewall) such as [ModSecurity](https://github.com/SpiderLabs/ModSecurity) can provide an additional detection and blocking layer against XXE payloads. WAFs should never be treated as the primary control since they can be bypassed. The back-end must be hardened independently.
