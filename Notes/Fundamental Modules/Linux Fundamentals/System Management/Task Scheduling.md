# Task Scheduling

Task scheduling allows you to **automate actions on a Linux system** so that they run at specific times or at regular intervals without manual input. You will encounter this functionality on almost every modern distribution, and it is used for tasks such as software updates, backups, maintenance scripts, and monitoring jobs 

You can think of task scheduling like setting an alarm or a coffee machine timer. Once configured, the system reliably performs the task for you at the defined time.

From a security perspective, task scheduling matters a lot. As a defender or penetration tester, you must understand how scheduled tasks work so you can:

* Identify unauthorised or malicious scheduled jobs
* Detect persistence mechanisms
* Analyse scripts that execute automatically with elevated privileges
* Simulate real-world attack scenarios safely

Linux primarily offers **two mechanisms** for task scheduling: **systemd timers** and **cron**.

---

## systemd Timers

Most modern Linux systems use **systemd** as their init and service manager. In addition to managing services, systemd can also schedule tasks using **timers**.

A systemd-based scheduled task consists of two components:

1. A **timer unit** – defines *when* something runs
2. A **service unit** – defines *what* is executed

The general workflow looks like this:

1. Create a timer
2. Create a service
3. Reload systemd
4. Start and enable the timer

---

## Creating a Timer

First, you create a location for the timer configuration and the timer file itself.

```text
sudo mkdir /etc/systemd/system/mytimer.timer.d
sudo vim /etc/systemd/system/mytimer.timer
```

### Example: `mytimer.timer`

```text
[Unit]
Description=My Timer

[Timer]
OnBootSec=3min
OnUnitActiveSec=1hour

[Install]
WantedBy=timers.target
```

What this means in practice:

* `OnBootSec=3min`
  → Run the task once, three minutes after boot

* `OnUnitActiveSec=1hour`
  → Re-run the task every hour after the previous execution

You choose these options based on your goal. If you only want something to run once after boot, `OnBootSec` is enough. If you want recurring execution, you use `OnUnitActiveSec`.

---

## Creating a Service

Next, you define **what actually runs** when the timer fires.

```text
sudo vim /etc/systemd/system/mytimer.service
```

### Example: `mytimer.service`

```text
[Unit]
Description=My Service

[Service]
ExecStart=/full/path/to/my/script.sh

[Install]
WantedBy=multi-user.target
```

Here:

* `ExecStart` must contain the **absolute path** to the script or command
* `multi-user.target` ensures the service runs in a standard multi-user environment

---

## Reloading systemd

After creating or modifying unit files, you must reload systemd so it becomes aware of the changes.

```text
sudo systemctl daemon-reload
```

---

## Starting and Enabling the Timer

Now you can start the timer and enable it so it persists across reboots.

```text
sudo systemctl start mytimer.timer
sudo systemctl enable mytimer.timer
```

From this point onward, `mytimer.service` will execute automatically according to the schedule defined in `mytimer.timer`.

---

## Cron

**Cron** is the traditional Linux task scheduler and is still widely used. It works by executing commands listed in a **crontab** file at specified times.

Instead of timers and services, cron relies on a simple time-based syntax that defines *when* a command runs.

---

## Cron Time Format

Each cron entry consists of **five time fields**, followed by the command to execute.

| Field        | Description                |
| ------------ | -------------------------- |
| Minute       | 0–59                       |
| Hour         | 0–23                       |
| Day of month | 1–31                       |
| Month        | 1–12                       |
| Day of week  | 0–7 (Sunday can be 0 or 7) |

---

## Example Crontab Entries

```text
# System update every 6 hours
0 */6 * * * /path/to/update_software.sh

# Run scripts on the first day of each month at midnight
0 0 1 * * /path/to/scripts/run_scripts.sh

# Clean database every Sunday at midnight
0 0 * * 0 /path/to/scripts/clean_database.sh

# Weekly backups every Sunday at midnight
0 0 * * 7 /path/to/scripts/backup.sh
```

How to read one of these:

```text
0 */6 * * *
```

* Minute: `0`
* Hour: every 6 hours
* Day, month, weekday: any

This means the task runs **every six hours on the hour**.

---

## Notifications and Logging

Cron can also:

* Send email notifications on success or failure
* Write output to log files
* Be combined with redirection to capture errors

This makes it useful for monitoring and auditing automated tasks.

---

## systemd vs Cron

Both tools achieve similar goals, but they differ in approach.

**systemd timers**:

* Event-based and time-based
* Integrated with service management
* More verbose and structured

**Cron**:

* Time-based only
* Simple and compact syntax
* Widely understood and quick to configure

In real environments, you will often see **both** used.

---

## Why This Matters

As you continue, always pay attention to scheduled tasks. They are a common place to find:

* Persistence mechanisms
* Misconfigured scripts running as root
* Hidden backdoors
* Maintenance jobs that can be abused

You should practise inspecting systemd timers and cron jobs, understanding what they execute, and analysing whether they are legitimate.

Task scheduling is not just automation. It is also **attack surface**.
