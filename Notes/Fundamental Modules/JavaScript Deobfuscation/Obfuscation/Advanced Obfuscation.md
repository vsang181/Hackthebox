# Advanced Obfuscation

At this stage, the code has been obfuscated and made harder to read. However, cleartext strings are still present, which can reveal aspects of the original functionality. In this section, additional tools are used to fully obfuscate the code and remove any visible traces of its original logic.

# Obfuscator

Navigate to [https://obfuscator.io](https://obfuscator.io). Before clicking the obfuscate option, change the String Array Encoding setting to Base64, as shown below:
JavaScript obfuscation tool with options for compact code, string array manipulation, and sourcemaps.

![js_deobf_obfuscator_2](https://github.com/user-attachments/assets/cc8ee27c-d226-409f-a0d0-62087e1bb3a0)

Next, paste the code into the input area and click obfuscate:
JavaScript code input area with Obfuscate button.

![js_deobf_obfuscator_1](https://github.com/user-attachments/assets/974b899f-6a3b-4aff-91c7-3ef048da8ef9)

The following output is generated:
Code: javascript

```
var _0x1ec6=['Bg9N','sfrciePHDMfty3jPChqGrgvVyMz1C2nHDgLVBIbnB2r1Bgu='];(function(_0x13249d,_0x1ec6e5){var _0x14f83b=function(_0x3f720f){while(--_0x3f720f){_0x13249d['push'](_0x13249d['shift']());}};_0x14f83b(++_0x1ec6e5);}(_0x1ec6,0xb4));var _0x14f8=function(_0x13249d,_0x1ec6e5){_0x13249d=_0x13249d-0x0;var _0x14f83b=_0x1ec6[_0x13249d];if(_0x14f8['eOTqeL']===undefined){var _0x3f720f=function(_0x32fbfd){var _0x523045='abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+/=',_0x4f8a49=String(_0x32fbfd)['replace'](/=+$/,'');var _0x1171d4='';for(var _0x44920a=0x0,_0x2a30c5,_0x443b2f,_0xcdf142=0x0;_0x443b2f=_0x4f8a49['charAt'](_0xcdf142++);~_0x443b2f&&(_0x2a30c5=_0x44920a%0x4?_0x2a30c5*0x40+_0x443b2f:_0x443b2f,_0x44920a++%0x4)?_0x1171d4+=String['fromCharCode'](0xff&_0x2a30c5>>(-0x2*_0x44920a&0x6)):0x0){_0x443b2f=_0x523045['indexOf'](_0x443b2f);}return _0x1171d4;};_0x14f8['oZlYBE']=function(_0x8f2071){var _0x49af5e=_0x3f720f(_0x8f2071);var _0x52e65f=[];for(var _0x1ed1cf=0x0,_0x79942e=_0x49af5e['length'];_0x1ed1cf<_0x79942e;_0x1ed1cf++){_0x52e65f+='%'+('00'+_0x49af5e['charCodeAt'](_0x1ed1cf)['toString'](0x10))['slice'](-0x2);}return decodeURIComponent(_0x52e65f);},_0x14f8['qHtbNC']={},_0x14f8['eOTqeL']=!![];}var _0x20247c=_0x14f8['qHtbNC'][_0x13249d];return _0x20247c===undefined?(_0x14f83b=_0x14f8['oZlYBE'](_0x14f83b),_0x14f8['qHtbNC'][_0x13249d]=_0x14f83b):_0x14f83b=_0x20247c,_0x14f83b;};console[_0x14f8('0x0')](_0x14f8('0x1'));
```

This output is significantly more obfuscated, and no readable elements from the original code remain. The script can be executed in [https://jsconsole.com](https://jsconsole.com) to confirm that it still performs its intended behaviour. Experimenting with different obfuscation settings in [https://obfuscator.io](https://obfuscator.io) can produce even more complex results, which can then be tested again to confirm functional consistency.

# More Obfuscation

At this point, the core concepts of code obfuscation should be clear. There are many variations of obfuscation tools, each applying different techniques. Consider the following JavaScript code:
Code: javascript

```
[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]][([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(!
...SNIP...
[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]]](!+[]+!+[]+[+[]])))()
```

This code can still be executed and will produce the same output as the original implementation:
Obfuscated JavaScript code output with HTB JavaScript Deobfuscation Module message.

![js_deobf_jsf](https://github.com/user-attachments/assets/3abd9770-295b-4866-9ddc-49c5674fb469)

Note: The code above has been truncated, as the full version is significantly longer. The complete script should still execute successfully.

The same obfuscation approach can be applied using tools such as JSF, after which the code can be executed again. It may be observed that execution time increases, demonstrating how obfuscation can negatively impact performance.

Many other JavaScript obfuscation techniques exist, such as JJ Encode and AA Encode. These methods often result in extremely slow execution or compilation times and are generally not recommended unless there is a specific requirement, such as bypassing web filters or enforced restrictions.
