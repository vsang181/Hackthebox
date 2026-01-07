# Deobfuscation

Now that the fundamentals of code obfuscation are understood, the focus can shift to deobfuscation. Just as there are tools that automatically obfuscate code, there are also tools designed to beautify and deobfuscate code automatically.

# Beautify

The current code is written on a single line, which is known as minified JavaScript. To properly analyse the code, it must first be formatted. The most basic way to achieve this is by using browser developer tools.

For example, in Firefox, the browser debugger can be opened using CTRL + SHIFT + Z. After selecting the script secret.js, the code is displayed in its original structure. By clicking the { } button at the bottom of the debugger, the script is pretty printed into readable JavaScript formatting:
Code editor showing JavaScript file secret.js with regex replacement code.

![js_deobf_pretty_print](https://github.com/user-attachments/assets/280f955d-1091-491a-a4ee-1fae3aa53f5a)

In addition to browser tools, many online services and editor plugins can be used, such as Prettier or Beautifier. The secret.js script can be copied as shown below:
Code: javascript

```
eval(function (p, a, c, k, e, d) { e = function (c) { return c.toString(36) }; if (!''.replace(/^/, String)) { while (c--) { d[c.toString(a)] = k[c] || c.toString(a) } k = [function (e) { return d[e] }]; e = function () { return '\\w+' }; c = 1 }; while (c--) { if (k[c]) { p = p.replace(new RegExp('\\b' + e(c) + '\\b', 'g'), k[c]) } } return p }('g 4(){0 5="6{7!}";0 1=8 a();0 2="/9.c";1.d("e",2,f);1.b(3)}', 17, 17, 'var|xhr|url|null|generateSerial|flag|HTB|flag|new|serial|XMLHttpRequest|send|php|open|POST|true|function'.split('|'), 0, {}))
```

Both approaches successfully reformat the code:
Code editor displaying obfuscated JavaScript function using eval.
Obfuscated JavaScript code using eval and regex replacement.

![js_deobf_prettier_1](https://github.com/user-attachments/assets/bc331991-33d3-44d4-b088-73b638fed709)

![js_deobf_beautifier_1](https://github.com/user-attachments/assets/97eddc10-8547-42a9-93dc-50cc1a9d0fd0)

However, the code remains difficult to interpret. This is because it has been obfuscated in addition to being minified. Formatting alone is insufficient, and dedicated deobfuscation techniques are required.

# Deobfuscate

Several online tools exist to deobfuscate JavaScript code and transform it into a more readable form. One such tool is UnPacker. The obfuscated script can be copied and processed by clicking the UnPack button.

Tip: Ensure there are no empty lines before the script, as this can interfere with the deobfuscation process and produce inaccurate results.
Obfuscated JavaScript code with functions for generating and sending serials.

![js_deobf_unpacker_1](https://github.com/user-attachments/assets/522626d5-cbc9-470a-8e7a-d01c77646095)

The output produced by the tool is significantly clearer and easier to understand:
Code: javascript

```
function generateSerial() {
  ...SNIP...
  var xhr = new XMLHttpRequest;
  var url = "/serial.php";
  xhr.open("POST", url, true);
  xhr.send(null);
};
```

As previously discussed, this form of obfuscation is known as packing. Another method of unpacking such code is to locate the return value at the end of the function and replace its execution with a console.log call to print the decoded result.

# Reverse Engineering

Although automated tools can effectively clean and simplify moderately obfuscated code, heavily obfuscated or encoded scripts are far more difficult to process automatically. This is particularly true when custom obfuscation techniques are used.

In such cases, manual reverse engineering is required to understand how the code was obfuscated and how it functions internally. For deeper coverage of advanced JavaScript deobfuscation and reverse engineering techniques, the Secure Coding 101 module provides a comprehensive exploration of these topics.
