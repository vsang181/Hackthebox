## Performance

When you are scanning large networks or working with limited bandwidth, **scan performance becomes just as important as scan accuracy**. Nmap provides several options that let you control how fast packets are sent, how long it waits for responses, and how persistent it is when a target does not reply.

Your goal is always to strike a balance between **speed**, **stealth**, and **accuracy**.

Key performance-related options you will work with include:

* Timing templates (`-T <0–5>`)
* Packet parallelism
* RTT timeouts
* Packet send rates
* Retry limits

---

## Timeouts (RTT Tuning)

Every packet Nmap sends has a **round-trip time (RTT)**: the time between sending a probe and receiving a response. If the timeout is too high, scans are slow. If it is too low, you risk missing hosts and open ports.

By default, Nmap starts with a relatively generous RTT value. You can tune this manually.

### Example: Default Scan

```
sudo nmap 10.129.2.0/24 -F
```

Result:

* Slower scan
* More reliable host detection

### Example: Optimised RTT

```
sudo nmap 10.129.2.0/24 -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms
```

This scan completes much faster but may miss hosts that respond slowly.

### Key takeaway

* Shorter RTT values = faster scans
* Too aggressive RTT tuning = false negatives
* Use conservative RTT values on unstable or high-latency networks

---

## Retry Behaviour

Nmap retries probes when it does not receive a response. By default, it retries **up to 10 times**, which improves accuracy but slows the scan.

You can reduce this behaviour:

```
--max-retries <number>
```

### Example: Default Retries

```
sudo nmap 10.129.2.0/24 -F
```

### Example: No Retries

```
sudo nmap 10.129.2.0/24 -F --max-retries 0
```

### Trade-off

* Fewer retries = faster scans
* Fewer retries = higher chance of missing filtered or slow services

This option is useful when speed matters more than completeness.

---

## Packet Rates

If you know the network can handle higher traffic (for example, during a **white-box assessment**), you can explicitly control how many packets Nmap sends per second.

```
--min-rate <number>
```

This tells Nmap to maintain a minimum packet send rate.

### Example: Default Rate

```
sudo nmap 10.129.2.0/24 -F -oN tnet.default
```

### Example: Increased Rate

```
sudo nmap 10.129.2.0/24 -F --min-rate 300 -oN tnet.fast
```

### Observations

* Scan time drops significantly
* Results remain accurate if the network is stable
* High packet rates increase detection risk on monitored networks

---

## Timing Templates (-T)

When you cannot manually tune every parameter, Nmap provides **timing templates** that bundle multiple performance settings together.

```
-T <0–5>
```

### Timing Levels

* `-T0` paranoid – extremely slow, highly stealthy
* `-T1` sneaky – very slow, low noise
* `-T2` polite – reduced bandwidth usage
* `-T3` normal – default behaviour
* `-T4` aggressive – faster scans, higher noise
* `-T5` insane – maximum speed, very noisy

### Example: Default Timing

```
sudo nmap 10.129.2.0/24 -F -oN tnet.default
```

### Example: Insane Timing

```
sudo nmap 10.129.2.0/24 -F -T5 -oN tnet.T5
```

Both scans may find the same open ports, but the aggressive scan completes much faster.

---

## Choosing the Right Performance Strategy

When tuning performance, always consider:

* Network size
* Bandwidth stability
* IDS/IPS presence
* Engagement scope (black-box vs white-box)

### Practical guidance

* Start conservative (`-T3`)
* Increase speed only when needed
* Avoid aggressive settings on unknown networks
* Validate fast scans with slower confirmation scans

Performance tuning is powerful, but misuse can cost you accuracy or get you detected.
