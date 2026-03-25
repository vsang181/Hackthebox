## Setting Up Burp Suite and ZAP

Both tools run on Windows, macOS, and Linux, and come pre-installed on penetration testing distributions like Kali and Parrot, as well as the HTB PwnBox. If you need to install them on a personal VM, the process is straightforward on either platform.

***

## Installing Burp Suite

Download the installer from portswigger.net/burp/releases and run it for your operating system. Alternatively, download the cross-platform JAR file and launch it with:

```bash
java -jar /path/to/burpsuite.jar
```

On Linux you can also launch it directly from the terminal:

```bash
burpsuite
```

### First Launch Options

When Burp starts, it prompts you to configure a project:

- **Community version**: Only temporary projects are available. Progress cannot be saved between sessions.
- **Pro/Enterprise version**: You can create, save, and reopen persistent projects, which is useful for long-running assessments or active scans.

For most training and standard engagements, a temporary project is fine. After selecting the project type, you are prompted to choose a configuration. Select "Use Burp Defaults" unless you have a custom configuration file saved from a previous setup.

***

## Installing ZAP

Download ZAP from zaproxy.org/download and run the appropriate installer, or launch the JAR file the same way as Burp:

```bash
java -jar /path/to/zap.jar
```

From the terminal once installed:

```bash
zaproxy
```

On first launch, ZAP also asks whether to create a persistent or temporary session. Unlike Burp Community, ZAP gives you this choice even in its free version. For most work, selecting "No" (temporary session) is sufficient unless you are running a multi-day assessment.

***

## Key Differences at Setup

| Factor | Burp Community | ZAP |
|--------|---------------|-----|
| Persistent projects | Not available | Available for free |
| First-launch prompt | Project type, then config | Session persistence only |
| Custom config loading | Supported | Supported via options menu |
| Terminal command | `burpsuite` | `zaproxy` |

***

## Dark Theme Setup

Both tools support dark themes, which is easier on the eyes during long sessions:

- **Burp**: `Burp > Settings > User Interface > Display` then set Theme to "Dark"
- **ZAP**: `Tools > Options > Display` then set Look and Feel to "Flat Dark"

***

## Java Dependency

Both Burp and ZAP depend on Java Runtime Environment (JRE) to run. Modern installers typically bundle a compatible JRE, so you usually do not need to install Java separately. If you use the standalone JAR files, ensure a compatible JRE is installed on the system beforehand. Running the JAR without a JRE installed will result in the command failing silently or throwing a `java: command not found` error.
