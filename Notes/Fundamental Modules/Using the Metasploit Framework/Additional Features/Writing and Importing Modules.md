# Writing and Importing Modules

When a required Metasploit module is absent from the current framework installation, there are two resolution paths: performing a full framework update via `apt`, or manually importing a specific module sourced from [ExploitDB](https://www.exploit-db.com/). The manual approach is preferable when only one module is needed and a full update is either unnecessary or would introduce breaking changes to a production assessment environment.

## Finding Modules on ExploitDB

ExploitDB's web interface supports tag-based filtering. Selecting the [Metasploit Framework (MSF)](https://www.exploit-db.com/?tag=3) tag restricts results to entries that have an accompanying Metasploit-compatible Ruby module. For command-line access to the same database, [searchsploit](https://www.exploit-db.com/searchsploit) is the preferred tool and is pre-installed on Kali Linux and Parrot OS Security:

```bash
searchsploit nagios3

Exploit Title                                                       |  Path
-------------------------------------------------------------------- | ---------------------------------
Nagios3 - 'history.cgi' Host Command Execution (Metasploit)         | linux/remote/24159.rb
Nagios3 - 'statuswml.cgi' 'Ping' Command Execution (Metasploit)     | cgi/webapps/16908.rb
Nagios3 - 'statuswml.cgi' Command Injection (Metasploit)            | unix/webapps/9861.rb
```

Results ending in `.rb` are Ruby scripts, which are likely Metasploit-compatible. However, `.rb` alone does not guarantee a valid Metasploit module; some Ruby exploits are standalone scripts without any Framework-compatible structure. The `(Metasploit)` label in the title is the reliable indicator. To exclude non-Ruby results and reduce noise:

```bash
searchsploit -t Nagios3 --exclude=".py"
```

`searchsploit` supports a range of useful flags:

| Flag | Function |
|---|---|
| `-t <term>` | Search only exploit titles rather than titles and file paths |
| `--exclude="<term>"` | Exclude results matching a string; chain multiple values with `\|` |
| `-m <EDB-ID>` | Mirror (copy) an exploit to the current working directory |
| `-p <EDB-ID>` | Show the full path to an exploit on the local system |
| `-x <EDB-ID>` | Open the exploit file in the system pager for inspection |
| `--cve <CVE>` | Search by CVE identifier |
| `--nmap <file.xml>` | Cross-reference Nmap XML version scan output against the exploit database |

## Installing a Module Manually

The Metasploit Framework stores all modules under `/usr/share/metasploit-framework/modules/`. A secondary path at `~/.msf4/modules/` is available for user-scoped modules and mirrors the same directory structure. Either location is valid for custom module installation.

After downloading the `.rb` file from ExploitDB, copy it into the appropriate subdirectory path. The subdirectory must match the module type and target platform:

```bash
cp ~/Downloads/9861.rb /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb
```

### Naming Conventions

Module filenames must follow strict naming rules or `msfconsole` will fail to load them:

- Use `snake_case` exclusively: lowercase alphanumeric characters and underscores only
- Never use dashes, spaces, or special characters in filenames
- The filename should describe the vulnerability clearly and match the module's `Name` field in spirit

```
# Correct
nagios3_command_injection.rb
bludit_auth_bruteforce_mitigation_bypass.rb

# Incorrect
nagios3-command-injection.rb
BluditAuth.rb
```

### Loading the Module

Three methods are available to make `msfconsole` recognise the newly installed module:

```bash
# Method 1: Launch msfconsole with an explicit module path
msfconsole -m /usr/share/metasploit-framework/modules/

# Method 2: Load a path into a running console session
msf6 > loadpath /usr/share/metasploit-framework/modules/

# Method 3: Reload all modules in a running console session
msf6 > reload_all
```

After loading, verify and use the module:

```bash
msf6 > use exploit/unix/webapp/nagios3_command_injection
msf6 exploit(unix/webapp/nagios3_command_injection) > show options

Module options:
   RHOSTS   (required)
   RPORT    80
   USER     guest
   PASS     guest
   URI      /nagios3/cgi-bin/statuswml.cgi
```

## Porting Existing Scripts to Metasploit Modules

When a working exploit exists only as a standalone Python, PHP, or Ruby script, it can be ported into a proper Metasploit module. The porting process does not start from a blank file. Instead, an existing module that operates against a similar service or vulnerability class is used as boilerplate, which provides the correct structure, mixin includes, and option registration patterns.

The general porting workflow is:

1. Identify an existing module in the same category as the target exploit to use as a template.
2. Adjust the `include` statements at the top of the file to match the mixins required by the new module.
3. Fill in the `initialize` block with the correct module metadata.
4. Replace the `register_options` block with the options required by the new exploit logic.
5. Port the exploit proof-of-concept logic into the `run` or `exploit` method, replacing standalone HTTP client calls with the Framework's built-in `HttpClient` mixin methods.

### Mixin Reference

The boilerplate Bludit Directory Traversal module includes four mixin statements. Each corresponds to a specific area of framework functionality:

| Mixin | Function |
|---|---|
| `Msf::Exploit::Remote::HttpClient` | Provides methods for acting as an HTTP client when exploiting an HTTP server |
| `Msf::Exploit::PhpEXE` | Generates first-stage PHP payloads |
| `Msf::Exploit::FileDropper` | Handles file transfer and post-session cleanup of dropped files |
| `Msf::Auxiliary::Report` | Provides methods for recording findings to the MSF database |

For the brute-force port-over, `FileDropper` is unnecessary as no files are written to the target. It is removed, and the `BLUDITPASS` string option is replaced with an `OptPath` that points to a wordlist:

```ruby
# Before: single password string
OptString.new('BLUDITPASS', [true, 'The password for Bludit'])

# After: wordlist path for brute-force
OptPath.new('PASSWORDS', [ true, 'The list of passwords',
  File.join(Msf::Config.data_directory, "wordlists", "passwords.txt") ])
```

### Module Header Structure

Every Metasploit module must follow this structural template:

```ruby
##
# This module requires Metasploit: https://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking

  include Msf::Exploit::Remote::HttpClient
  include Msf::Auxiliary::Report

  def initialize(info={})
    super(update_info(info,
      'Name'           => "Bludit 3.9.2 - Authentication Bruteforce Mitigation Bypass",
      'Description'    => %q{ ... },
      'License'        => MSF_LICENSE,
      'Author'         => [ 'rastating', '0ne-nine9' ],
      'References'     =>
        [
          ['CVE', '2019-17240'],
          ['URL', 'https://rastating.github.io/bludit-brute-force-mitigation-bypass/']
        ],
      'Platform'       => 'php',
      'Arch'           => ARCH_PHP,
      'Targets'        => [ [ 'Bludit v3.9.2', {} ] ],
      'Privileged'     => false,
      'DisclosureDate' => "2019-10-05",
      'DefaultTarget'  => 0))

    register_options(
      [
        OptString.new('TARGETURI', [true, 'The base path for Bludit', '/']),
        OptString.new('BLUDITUSER', [true, 'The username for Bludit']),
        OptPath.new('PASSWORDS', [ true, 'The list of passwords',
          File.join(Msf::Config.data_directory, "wordlists", "passwords.txt") ])
      ])
  end

  def exploit
    # Exploit logic here
  end
end
```

Key points when filling in the header:

- `Rank` values follow the standard scale: `ExcellentRanking`, `GreatRanking`, `GoodRanking`, `NormalRanking`, `AverageRanking`, `LowRanking`, `ManualRanking`
- The `References` array accepts `CVE`, `URL`, `EDB`, `BID`, and `PATCH` reference types
- `Notes` should include `SideEffects`, `Reliability`, and `Stability` where applicable
- All Ruby module code uses hard tabs for indentation, not spaces

Full documentation for available mixins, classes, and method signatures is available in the [Metasploit API documentation](https://docs.metasploit.com/api/). The [Metasploit: A Penetration Tester's Guide](https://nostarch.com/metasploit) from No Starch Press and Rapid7's [module development blog series](https://www.rapid7.com/blog/post/2012/07/05/part-1-metasploit-module-development-the-series/) are the most thorough external references for deeper module development work.
