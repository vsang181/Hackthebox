# Intro to XXE

XML External Entity (XXE) Injection occurs when an application parses XML input from a user without properly sanitising or safely handling it. This allows attackers to abuse XML features to carry out malicious actions. XXE can cause serious damage ranging from disclosing sensitive files to taking down a back-end server entirely. [OWASP](https://owasp.org/www-project-top-ten/) lists it among the Top 10 Web Security Risks.

***

## XML Basics

Extensible Markup Language (XML) is a markup language designed for flexible data storage and transfer across different types of applications. Unlike HTML, XML focuses on representing and storing data structures rather than displaying them. XML documents are built as element trees, where the first element is the root and all others are child elements.

Here is a basic XML document representing an email structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<email>
  <date>01-01-2022</date>
  <time>10:00 am UTC</time>
  <sender>john@inlanefreight.com</sender>
  <recipients>
    <to>HR@inlanefreight.com</to>
    <cc>
        <to>billing@inlanefreight.com</to>
        <to>payslips@inlanefreight.com</to>
    </cc>
  </recipients>
  <body>
  Hello,
      Kindly share with me the invoice for the payment made on January 1, 2022.
  Regards,
  John
  </body>
</email>
```

### Key XML Components

| Key | Definition | Example |
|---|---|---|
| Tag | The keys of an XML document, wrapped with `</>` characters | `<date>` |
| Entity | XML variables, wrapped with `&/;` characters | `&lt;` |
| Element | The root or any child element, with its value stored between a start-tag and end-tag | `<date>01-01-2022</date>` |
| Attribute | Optional specifications stored inside tags, used by the XML parser | `version="1.0"` / `encoding="UTF-8"` |
| Declaration | Usually the first line of an XML document, defines version and encoding | `<?xml version="1.0" encoding="UTF-8"?>` |

Certain characters such as `<`, `>`, `&`, and `"` form part of the XML structure itself. When you need to use them as data, replace them with their entity references: `&lt;`, `&gt;`, `&amp;`, and `&quot;`. XML comments are written between `<!--` and `-->`, similar to HTML.

***

## XML DTD

[XML Document Type Definition (DTD)](https://www.w3schools.com/xml/xml_dtd_intro.asp) allows an XML document to be validated against a pre-defined structure. That structure can live inside the document itself or in a separate external file.

```xml
<!DOCTYPE email [
  <!ELEMENT email (date, time, sender, recipients, body)>
  <!ELEMENT recipients (to, cc?)>
  <!ELEMENT cc (to*)>
  <!ELEMENT date (#PCDATA)>
  <!ELEMENT time (#PCDATA)>
  <!ELEMENT sender (#PCDATA)>
  <!ELEMENT to  (#PCDATA)>
  <!ELEMENT body (#PCDATA)>
]>
```

The DTD declares the root `email` element using `ELEMENT` type declarations, then defines each child element. Elements marked `#PCDATA` contain raw text data. The DTD can be placed directly in the XML document after the declaration, or saved as an external file and referenced with the `SYSTEM` keyword:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email SYSTEM "email.dtd">
```

You can also reference a DTD through a URL:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email SYSTEM "http://inlanefreight.com/email.dtd">
```

***

## XML Entities

Custom entities act as XML variables and allow you to reduce repetitive data in a document. You define them using the `ENTITY` keyword inside a DTD block:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

Once defined, reference the entity anywhere in the document using `&company;`. The XML parser replaces every reference with the entity's value at parse time.

### External Entities

You can point entities at external resources using the `SYSTEM` keyword:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "http://localhost/company.txt">
  <!ENTITY signature SYSTEM "file:///var/www/html/signature.txt">
]>
```

When the parser encounters `&signature;`, it fetches the content of the referenced file and substitutes it in place. In server-side XML parsing scenarios such as [SOAP](https://www.w3schools.com/xml/xml_soap.asp) APIs or web forms, this means an external entity can reference files stored on the back-end server. If that parsed output is returned to the user, the file contents get disclosed.

- Use `SYSTEM` for local or private external resources
- Use `PUBLIC` for publicly declared resources and standards such as language codes
- Either keyword works in most practical cases, but this module uses `SYSTEM`
