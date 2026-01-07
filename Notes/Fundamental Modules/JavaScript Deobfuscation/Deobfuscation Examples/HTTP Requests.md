# HTTP Requests

In the previous section, it was identified that the main function in secret.js sends an empty POST request to /serial.php. In this section, the same behaviour is replicated using cURL to send a POST request to /serial.php. To learn more about cURL and web requests, the Web Requests module can be referenced.

# cURL

cURL is a powerful command-line tool available on Linux distributions, macOS, and recent versions of Windows PowerShell. It allows requests to be sent to web servers by specifying a URL, returning the response in text format, as shown below:
HTTP Requests

```
willywonka69@htb[/htb]$ curl http://SERVER_IP:PORT/

</html>
<!DOCTYPE html>

<head>
    <title>Secret Serial Generator</title>
    <style>
        *,
        html {
            margin: 0;
            padding: 0;
            border: 0;
...SNIP...
        <h1>Secret Serial Generator</h1>
        <p>This page generates secret serials!</p>
    </div>
</body>

</html>
```

This response contains the same HTML that was previously examined when viewing the page source.

# POST Request

To send a POST request, the -X POST flag can be added to the command, which instructs cURL to use the POST method:
HTTP Requests

```
willywonka69@htb[/htb]$ curl -s http://SERVER_IP:PORT/ -X POST
```

Tip: The -s flag is used to suppress unnecessary output and reduce clutter in the response.

POST requests typically include data. To send data, the -d flag can be used to define one or more parameters, as shown below:
HTTP Requests

```
willywonka69@htb[/htb]$ curl -s http://SERVER_IP:PORT/ -X POST -d "param1=sample"
```

With this understanding of how to send POST requests using cURL, the next section will use this approach to replicate the behaviour of server.js in order to better understand its purpose.
