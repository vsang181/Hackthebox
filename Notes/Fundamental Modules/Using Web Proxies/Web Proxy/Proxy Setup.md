## Proxy Setup

Getting a web proxy working with your browser comes down to two things: configuring the browser to route traffic through the proxy listener, and installing the proxy's CA certificate so HTTPS traffic decrypts cleanly without constant SSL warnings.

***

## Pre-Configured Browser (Quickest Start)

Both tools ship with a pre-configured browser that handles proxy settings and certificate trust automatically, making it the fastest way to get started:

- **Burp**: `Proxy > Intercept > Open Browser`
- **ZAP**: Click the Firefox icon at the right end of the top toolbar

This is sufficient for most module work and basic assessments. The browser launched this way already trusts the proxy's CA and routes all traffic through the listener without any additional configuration.

***

## Manual Firefox Proxy Setup

When you need to use your main Firefox installation, you have two options for pointing it at the proxy.

### Option 1: FoxyProxy Extension (Recommended)

[FoxyProxy](https://addons.mozilla.org/en-US/firefox/addon/foxyproxy-standard/) lets you switch between proxy profiles with a single click rather than digging into Firefox settings each time:

1. Install FoxyProxy Standard from the Firefox Extensions page (pre-installed on PwnBox)
2. Click the FoxyProxy icon in the top bar and select Options
3. Click Add in the left pane and fill in:
   - IP: `127.0.0.1`
   - Port: `8080`
   - Name: Burp or ZAP
4. Save the profile
5. Click the FoxyProxy icon and select the profile to activate it

### Option 2: Manual Firefox Settings

Navigate to `about:preferences`, search for "proxy", and configure a manual proxy pointing at `127.0.0.1:8080`. This works but requires you to disable it manually each time you are done testing to restore normal browsing.

***

## Changing the Default Port

If port 8080 is already in use on your system, change the listener port before starting:

- **Burp**: `Proxy > Proxy Settings > Proxy Listeners`
- **ZAP**: `Tools > Options > Network > Local Servers/Proxies`

Whatever port the listener binds to must match the port configured in FoxyProxy or Firefox, otherwise traffic will not route through.

***

## Installing CA Certificates

Without the proxy's CA certificate installed in Firefox, HTTPS interception will either fail silently or produce a browser warning on every single HTTPS request. This step is not optional for any real testing work.

### Burp Certificate

1. With Burp set as your active proxy, browse to `http://burp`
2. Click "CA Certificate" to download the file
3. In Firefox go to `about:preferences#privacy`
4. Scroll to the bottom and click "View Certificates"
5. Select the "Authorities" tab, click "Import", and select the downloaded certificate
6. Check both "Trust this CA to identify websites" and "Trust this CA to identify email users"
7. Click OK

### ZAP Certificate

1. In ZAP go to `Tools > Options > Network > Server Certificates`
2. Click "Save" to export the certificate to disk
3. Follow the same Firefox import steps described above

You can regenerate a fresh ZAP certificate at any time using the "Generate" button in the same menu, which is useful if a certificate becomes outdated or compromised.

***

## Full Setup Flow

```
Install FoxyProxy
        |
Add proxy profile (127.0.0.1:8080)
        |
Start Burp or ZAP
        |
Activate profile in FoxyProxy
        |
Download and install CA cert in Firefox
        |
Browse to target — all traffic now flows through the proxy
```

Once the certificate is trusted and the proxy profile is active, every HTTP and HTTPS request Firefox makes will pass through the proxy. You can confirm everything is working by loading any site and checking that requests appear in Burp's HTTP history tab or ZAP's History tab at the bottom of the interface.
