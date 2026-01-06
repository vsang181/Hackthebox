# Working with Web Services

Communication between a browser and a web server is a foundational concept you must understand when working with Linux systems and web technologies. Web servers are responsible for receiving requests, processing them, and returning responses to clients. On Linux, this can be achieved using several different server implementations, with **Apache**, **Nginx**, and **IIS** being some of the most common. Among these, Apache remains one of the most widely deployed and encountered in real-world environments 

You can think of a web server as the engine behind a website. It does not just serve files; it controls how requests are handled, how data flows, and how content is delivered to users.

---

## Apache Web Server

Apache is highly modular, which is one of its biggest strengths. Instead of being a single monolithic application, Apache can be extended using modules that add or modify behaviour.

A useful mental model is to think of Apache like a building framework:

* The **core server** is the foundation
* **Modules** act like extensions or rooms added for specific purposes

Some commonly used modules include:

* **`mod_ssl`** – encrypts traffic using TLS
* **`mod_proxy`** – forwards requests to other services
* **`mod_headers`** – manipulates HTTP headers
* **`mod_rewrite`** – rewrites URLs dynamically

These modules allow Apache to act not just as a simple file server, but as a reverse proxy, application gateway, or security control point.

Apache also supports **dynamic content generation** through server-side scripting languages such as PHP, Perl, and Ruby. Other languages, including Python, Lua, JavaScript, and even .NET, can also be integrated depending on the configuration. This is what enables modern, interactive web applications.

---

## Installing Apache

If Apache is not already installed, you can install it using the package manager:

```text
sudo apt install apache2 -y
```

Once installed, Apache can be managed using standard service management tools.

---

## Starting and Accessing Apache

You can start the Apache service using:

```text
sudo systemctl start apache2
```

By default, Apache listens on **TCP port 80** for HTTP traffic. If you open a browser and navigate to:

```text
http://localhost
```

you should see the default Apache page. This confirms that the server is running correctly and able to serve content.

---

## Changing the Listening Port

In some environments, such as shared lab systems or testing boxes, port 80 may already be in use. If this happens, you can change the port Apache listens on.

Edit the ports configuration file:

```text
sudo nano /etc/apache2/ports.conf
```

Example configuration:

```text
Listen 8080

<IfModule ssl_module>
Listen 443
</IfModule>
```

After making changes, restart Apache:

```text
sudo systemctl restart apache2
```

You can now access the server at:

```text
http://localhost:8080
```

---

## Verifying the Server with curl

You do not need a browser to interact with a web server. Tools like `curl` allow you to communicate with web services directly from the terminal.

To request only HTTP headers:

```text
curl -I http://localhost:8080
```

Example response:

```text
HTTP/1.1 200 OK
Server: Apache/2.4.62 (Debian)
Content-Type: text/html
```

This confirms that the server is responding and provides useful metadata such as the server version.

---

## Interacting with Web Servers from the Terminal

### curl

`curl` is a versatile command-line tool for transferring data over many protocols, including HTTP, HTTPS, FTP, and SCP. It is installed by default on most Linux systems and is indispensable during security assessments.

Fetching a webpage:

```text
curl http://localhost
```

This returns the **raw HTML source** of the page to STDOUT. Unlike a browser, `curl` does not render HTML, CSS, or JavaScript. This makes it ideal for inspecting responses, headers, and server behaviour directly.

---

### wget

`wget` serves a similar purpose but behaves differently. Instead of printing content to the terminal, it downloads files and saves them locally.

```text
wget http://localhost
```

This downloads the page and stores it as `index.html` in the current directory. `wget` is often used as a lightweight download manager when retrieving files during assessments.

---

## Hosting Files with Python

For quick and temporary file hosting, Python provides a built-in web server. This is extremely useful when you need to transfer files without installing or configuring a full web server.

Start a Python web server in the current directory:

```text
python3 -m http.server
```

By default, it listens on port **8000** and serves the current directory as the web root.

If you open a browser and navigate to:

```text
http://localhost:8000
```

you will see a directory listing or a file if one exists (for example, `readme.html`).

---

## Observing Requests

The Python web server logs incoming requests directly to the terminal:

```text
127.0.0.1 - - [15/May/2020 17:56:29] "GET /readme.html HTTP/1.1" 200 -
```

This gives you immediate visibility into which resources are being requested, from where, and with what result. This is extremely helpful when testing file access, debugging behaviour, or understanding how clients interact with a service.

---

## Learning Through Exploration

When working with web services, you will regularly encounter situations you have never seen before. This is expected.

Penetration testing is not about memorising commands. It is about:

* Researching unfamiliar behaviour
* Testing assumptions
* Trying multiple approaches
* Learning from mistakes

This is a learning process, not an exam. The challenges you face are intentionally designed to push you beyond your comfort zone. By experimenting, breaking things, and fixing them again, you develop the adaptability required for real-world environments.

Embrace curiosity. Explore configurations. Test different tools. The ability to learn independently and think creatively is one of the most important skills you can develop as you move forward.
