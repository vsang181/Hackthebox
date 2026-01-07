# Basic Obfuscation

Code obfuscation is typically not performed manually, as there are many tools available for different programming languages that automate the obfuscation process. Numerous online tools exist for this purpose, although many malicious actors and professional developers create their own obfuscation tools to make deobfuscation more difficult.

# Running JavaScript Code

Consider the following line of code as an example, which will be obfuscated:
Code: javascript

```
console.log('HTB JavaScript Deobfuscation Module');
```

First, the code is executed in cleartext to observe its behaviour. This can be done by opening JSConsole, pasting the code, and pressing enter to view the output:
Console output showing HTB JavaScript Deobfuscation Module log message and version 2.1.2.

![js_deobf_jsconsole_1_1](https://github.com/user-attachments/assets/99359061-c159-436c-89da-5e8a8e7265a0)

This line of code prints the text HTB JavaScript Deobfuscation Module using the console.log() function.

# Minifying JavaScript Code

A common method of reducing the readability of JavaScript code while preserving its functionality is JavaScript minification. Minification involves compressing the entire code into a single, often very long, line. This technique is more effective for larger scripts, as a single-line script does not change significantly when minified.

Several tools are available to minify JavaScript code, such as javascript-minifier. The process involves copying the original code, selecting the Minify option, and obtaining the minified output:
JavaScript minification tool showing input code and minified output.

![js_minify_1](https://github.com/user-attachments/assets/f66341e8-04f2-40cc-b351-5c99ee3dc9b8)

The minified code can be copied into JSConsole and executed, where it behaves identically to the original version. Minified JavaScript files are typically saved with the .min.js extension.

Note: Code minification is not limited to JavaScript and can be applied to many other programming languages.

# Packing JavaScript Code

The next step is to obfuscate the code further to make it more difficult to read. One approach is to use BeautifyTools to obfuscate the code:
JavaScript deobfuscation tool with options for fast decode and special characters, showing obfuscated code.
Code: javascript

![js_deobf_obfuscator](https://github.com/user-attachments/assets/073409fb-9501-449b-987a-d1bc45eb1fbd)

```
eval(function(p,a,c,k,e,d){e=function(c){return c};if(!''.replace(/^/,String)){while(c--){d[c]=k[c]||c}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('5.4(\'3 2 1 0\');',6,6,'Module|Deobfuscation|JavaScript|HTB|log|console'.split('|'),0,{}))
```

The code is now significantly more obfuscated and difficult to read. This obfuscated version can be copied into [https://jsconsole.com](https://jsconsole.com) to verify that it still performs its intended function:
Console output showing obfuscated JavaScript code and version 2.1.2.

![js_deobf_jsconsole_3_1(1)](https://github.com/user-attachments/assets/18699802-5c32-43b1-95a8-67e9c7540068)

The output remains the same as before.

Note: This type of obfuscation is known as packing and is commonly identifiable by the six function arguments used in the initial function definition, function(p,a,c,k,e,d).

A packer obfuscation tool typically converts words and symbols from the original code into a list or dictionary, then reconstructs the original logic during execution using the function arguments. While the specific parameters may vary between packers, they usually follow a structured order that allows the code to be reassembled correctly at runtime.

Although packing significantly reduces code readability, key strings often remain visible in cleartext, which can still reveal aspects of the script’s functionality. For this reason, more advanced obfuscation techniques may be required.
