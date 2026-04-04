# Tomcat Discovery and Enumeration

[Apache Tomcat](https://tomcat.apache.org/) is an open-source Java web server that hosts Java Servlet and [Jakarta Server Pages (JSP)](https://en.wikipedia.org/wiki/Jakarta_Server_Pages) applications. It is widely used with frameworks like [Spring](https://spring.io/) and build tools like [Gradle](https://gradle.org/). While less common on external perimeters than other web applications, Tomcat appears frequently during internal penetration tests and almost always occupies the top spot in [EyeWitness](https://github.com/FortyNorthSecurity/EyeWitness) reports under High Value Targets. Internal instances are often configured with default or weak credentials, making them reliable footholds.

***

## Quick Identification

Tomcat reveals itself through several indicators:

- The `Server` HTTP response header often contains the Tomcat version
- Requesting an invalid path returns a Tomcat-branded error page with version information
- The `/docs` page is often left in place and leaks the version in the page title:

```bash
curl -s http://app-dev.inlanefreight.local:8080/docs/ | grep Tomcat
```

```
<title>Apache Tomcat 9 (9.0.30) - Documentation Index</title>
```

***

## Tomcat Directory Structure

Understanding the file layout helps you know where to look for sensitive data:

```
├── bin                   # Scripts and binaries to start/run Tomcat
├── conf
│   ├── catalina.policy
│   ├── catalina.properties
│   ├── context.xml
│   ├── tomcat-users.xml  # User credentials and role assignments
│   ├── tomcat-users.xsd
│   └── web.xml
├── lib                   # JAR files required for Tomcat operation
├── logs                  # Runtime log files
├── temp                  # Temporary files
├── webapps               # Default webroot, hosts all deployed applications
│   ├── manager           # Manager application
│   └── ROOT
└── work                  # Runtime cache
    └── Catalina
        └── localhost
```

### Web Application Structure

Each application deployed under `webapps/` follows this layout:

```
webapps/customapp
├── index.jsp
├── META-INF
│   └── context.xml
└── WEB-INF
    ├── web.xml           # Deployment descriptor, defines routes and servlet mappings
    ├── jsp
    │   └── admin.jsp
    ├── lib
    │   └── jdbc_drivers.jar
    └── classes
        └── AdminServlet.class
```

The `WEB-INF/web.xml` deployment descriptor is the most important file in any Tomcat application. It maps URL routes to servlet classes and often contains sensitive configuration. An example:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<web-app>
  <servlet>
    <servlet-name>AdminServlet</servlet-name>
    <servlet-class>com.inlanefreight.api.AdminServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>AdminServlet</servlet-name>
    <url-pattern>/admin</url-pattern>
  </servlet-mapping>
</web-app>
```

This configuration maps all requests to `/admin` to the `AdminServlet.class` file, which would be located on disk at:

```
classes/com/inlanefreight/api/AdminServlet.class
```

`web.xml` is a high-priority target when exploiting a Local File Inclusion vulnerability, as it reveals the full application structure and servlet class names.

***

## tomcat-users.xml

The `conf/tomcat-users.xml` file controls access to the `/manager` and `/host-manager` admin panels. It defines user accounts and their assigned roles. The four available manager roles are:

| Role | Access |
|---|---|
| `manager-gui` | HTML GUI and status pages |
| `manager-script` | HTTP API and status pages |
| `manager-jmx` | JMX proxy and status pages |
| `manager-status` | Status pages only |

A poorly secured example of this file looks like:

```xml
<role rolename="manager-gui" />
<user username="tomcat" password="tomcat" roles="manager-gui" />

<role rolename="admin-gui" />
<user username="admin" password="admin" roles="manager-gui,admin-gui" />
```

Both accounts use default or trivially weak passwords. This file is commonly left unchanged on internal Tomcat deployments and is the primary reason Tomcat is such a reliable attack target during internal assessments.

***

## Enumeration

Once you have confirmed a Tomcat installation, the primary goal is locating the `/manager` and `/host-manager` endpoints. Browse directly to them or use [Gobuster](https://github.com/OJ/gobuster) to discover them:

```bash
gobuster dir -u http://web01.inlanefreight.local:8180/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

```
/docs    (Status: 302)
/examples (Status: 302)
/manager  (Status: 302)
```

Common default credentials to try against the manager login before brute-forcing:

- `tomcat:tomcat`
- `admin:admin`
- `admin:password`
- `tomcat:s3cr3t`

If the default credentials do not work, a brute-force attack against the login page is the next step. Gaining access to the `/manager` panel is the primary objective because it allows uploading a [WAR](https://en.wikipedia.org/wiki/WAR_(file_format)) file, which can contain a JSP web shell and deliver remote code execution on the underlying host.
