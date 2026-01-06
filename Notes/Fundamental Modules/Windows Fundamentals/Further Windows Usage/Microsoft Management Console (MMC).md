# Microsoft Management Console (MMC)

The **Microsoft Management Console (MMC)** is a framework you use to manage **hardware, software, and network components** on Windows systems. It has existed since **Windows Server 2000** and is available on all modern Windows versions. If you work with Windows long enough, you will encounter MMC constantly—sometimes directly, sometimes through tools that rely on it behind the scenes.

You should think of MMC as a **container**, not a tool by itself.

---

## What MMC Actually Does

MMC does not perform administration on its own. Instead, it hosts **snap-ins**, which are individual administrative tools.

Each snap-in is responsible for managing a specific component, such as:

* Services
* Event logs
* Local users and groups
* Device Manager
* Disk management
* Active Directory (on domain systems)

By combining snap-ins, you create a **custom administrative console** tailored to exactly what you need to manage.

---

## Why MMC Matters to You

MMC is important because it allows you to:

* Centralise administrative tools in one place
* Manage **local or remote systems**
* Reduce clutter by loading only required snap-ins
* Save reusable management configurations

From a security perspective, MMC access often implies **administrative privileges**, which makes it highly relevant during post-exploitation and lateral movement.

---

## Launching MMC

You can open MMC by typing the following in the Start menu or Run dialogue:

```text
mmc
```

When MMC opens for the first time, it is completely **empty**. You will only see:

* **Console Root** on the left
* An **Actions** pane on the right

This is expected behaviour.

---

## Adding Snap-ins

To customise MMC, you add snap-ins manually.

From the menu:

* Go to **File**
* Select **Add or Remove Snap-ins**

This opens a list of available snap-ins that you can load into the console.

---

## Local vs Remote Management

When adding many snap-ins, Windows will ask whether you want to manage:

* **The local computer**
* **Another computer on the network**

This distinction is important.

Managing remote systems through MMC requires:

* Network connectivity
* Appropriate permissions
* Sometimes firewall rule adjustments

As a penetration tester, remote MMC access is a strong indicator of **high trust** between systems.

---

## Working with Snap-ins

Once added, snap-ins appear in the **left-hand pane** under Console Root.

From here, you can:

* Expand nodes
* Inspect configuration
* Start or stop services
* Review logs
* Modify system settings

MMC provides a structured, hierarchical view of the system, which makes complex environments easier to navigate.

---

## Saving MMC Configurations

After you finish configuring your console, you can save it as an **`.msc` file**.

This file stores:

* The selected snap-ins
* Their configuration
* Target systems (local or remote)

By default, saved consoles are stored in the **Windows Administrative Tools** directory and appear in the Start menu.

The next time you open MMC, you can simply load your saved `.msc` file instead of rebuilding the console from scratch.

---

## Custom MMC Consoles in Practice

Administrators often create custom MMC consoles to:

* Delegate limited administrative tasks
* Reduce accidental misconfiguration
* Provide junior staff with controlled access
* Standardise system management across teams

From an offensive perspective, finding custom `.msc` files can reveal:

* What systems an administrator manages
* Which services are considered important
* How responsibilities are delegated

---

## Key Takeaways

* MMC is a **framework**, not a single tool
* All functionality comes from **snap-ins**
* You can manage **local and remote systems**
* Custom consoles can be saved as `.msc` files
* MMC access often implies elevated privileges

Once you understand MMC, many Windows administrative actions suddenly make more sense. It is one of the clearest windows into how administrators actually interact with Windows systems—and that insight is valuable both for defence and attack.
