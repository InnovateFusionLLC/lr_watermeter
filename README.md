# lr_watermeter
# Neptune R900 Water Meter Monitoring on macOS with RTL-SDR

This guide documents how to use an RTL-SDR on macOS to:

- Read your Neptune R900 water meter broadcasts (potable and irrigation)
- Decode them in real-time
- Identify and label each meter
- Track consumption automatically
- Log usage data for later analysis

---

## Hardware & Software Requirements

### Hardware
- RTL-SDR USB dongle (RTL2832U + R820T2 recommended)
- A simple whip antenna or tuned 915 MHz antenna (Neptune meters broadcast in this band)

### Software (macOS)
Installed via Homebrew and direct builds:
- `rtl-sdr` (rtl_tcp, rtl_test, rtl_power)
- `go` (for building rtlamr)
- `gnuplot` (optional for spectrum visualization)
- Python 3 (comes standard on macOS)

---

## Installation

### Install Homebrew (if needed)
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install required packages
```bash
brew install rtl-sdr go gnuplot
```

---

## Build rtlamr

Clone and build:
```bash
cd ~/Downloads
git clone https://github.com/bemasher/rtlamr.git
cd rtlamr
go build
```

This will produce an executable `./rtlamr` in this directory.

---

## Verify RTL-SDR is working

Run this command to confirm your RTL-SDR is detected:
```bash
rtl_test -t
```
Expected output:
```
Found 1 device(s):
  0: Realtek, RTL2838UHIDIR, SN: SDR00012
...
```

---

## (Optional) Scan for Neptune meter RF signals

You can use `rtl_power` to confirm the presence of your water meters in the ISM band:
```bash
rtl_power -f 902M:928M:50k -i 10 -e 2m scan.csv
```

Then plot with gnuplot:
```bash
gnuplot -e "set datafile separator ','; plot 'scan.csv' using 3:7 with lines"
```

Look for strong signals around **915 MHz**, typical for Neptune R900.

---

## Start decoding your meters

### Start rtl_tcp
In one terminal window:
```bash
rtl_tcp -a 127.0.0.1 -p 1234
```
Leave this running. This streams data from your RTL-SDR.

---

### In another terminal, decode with rtlamr
```bash
./rtlamr -msgtype=r900
```

This will start showing raw JSON-like outputs such as:
```
{Time:..., R900:{ID: 702929978, Consumption: ...}}
```

---

## Identifying your potable vs irrigation meters

We performed controlled tests to identify:
- `702929978` → Irrigation (large deltas when irrigation runs)
- `701193108` → Potable house water (smaller deltas when running sinks)
- `701187112` → Likely a neighbor’s meter (barely moves)

---

## Live tracker script

This repository includes a Python script that:
- Labels your known meters
- Tracks consumption deltas in real-time
- Logs everything to `meter_tracking_YYYY-MM-DD_HH-MM-SS.log` for analysis

---

### track_meters.py

This script watches the output from `rtlamr` and prints labeled results.  
It also logs everything to a timestamped file.

```python
#!/usr/bin/env python3
import json
import sys
from datetime import datetime

meter_ids = {
    "702929978": "IRRIGATION",
    "701193108": "POTABLE",
    "701187112": "LIKELY NEIGHBOR"
}

previous_values = {}
logfile_name = f"meter_tracking_{datetime.now().strftime('%Y-%m-%d_%H-%M-%S')}.log"
logfile = open(logfile_name, "w")

print(f"Logging updates to {logfile_name}")

try:
    for line in sys.stdin:
        line = line.strip()
        if not line.startswith("{"):
            continue
        try:
            line_fixed = line.replace("Time:", "\"Time\":") \
                             .replace("R900:{", "\"R900\":{") \
                             .replace("ID:", "\"ID\":") \
                             .replace("Unkn1:", "\"Unkn1\":") \
                             .replace("NoUse:", "\"NoUse\":") \
                             .replace("BackFlow:", "\"BackFlow\":") \
                             .replace("Consumption:", "\"Consumption\":") \
                             .replace("Unkn3:", "\"Unkn3\":") \
                             .replace("Leak:", "\"Leak\":") \
                             .replace("LeakNow:", "\"LeakNow\":")
            data = json.loads(line_fixed)
        except json.JSONDecodeError:
            continue

        meter_data = data.get("R900", {})
        meter_id = str(meter_data.get("ID"))
        consumption = meter_data.get("Consumption", 0)

        label = meter_ids.get(meter_id, f"UNRECOGNIZED ID {meter_id}")
        prev = previous_values.get(meter_id, consumption)
        delta = consumption - prev

        msg = f"[{label}] Consumption: {consumption:,} (∆{delta:+})"
        print(msg)
        logfile.write(msg + "\n")
        logfile.flush()

        previous_values[meter_id] = consumption

except KeyboardInterrupt:
    print("\nStopping meter tracking...")
finally:
    logfile.close()
```

---

## Running the tracker

Make it executable:
```bash
chmod +x track_meters.py
```

Then run it by piping `rtlamr` output:
```bash
./rtlamr -msgtype=r900 | ./track_meters.py
```

It prints live updates such as:
```
[IRRIGATION] Consumption: 2,660,489 (∆+7)
[POTABLE] Consumption: 6,056,951 (∆+1)
[LIKELY NEIGHBOR] Consumption: 4,665,995 (∆+0)
```

and also logs to `meter_tracking_YYYY-MM-DD_HH-MM-SS.log`.

---

## Optional: all-in-one shell script

This starts `rtl_tcp`, waits, then pipes into your Python tracker:

### log_water_meter.sh
```bash
#!/bin/bash
OUTPUT_DIR=~/rtlamr_logs
mkdir -p "$OUTPUT_DIR"
NOW=$(date +"%Y-%m-%d_%H-%M-%S")
LOGFILE="$OUTPUT_DIR/meter_log_$NOW.txt"

echo "Starting rtl_tcp..."
rtl_tcp -a 127.0.0.1 -p 1234 > "$OUTPUT_DIR/rtl_tcp_$NOW.txt" 2>&1 &
RTL_PID=$!
sleep 4
echo "Starting rtlamr + tracker..."
./rtlamr -msgtype=r900 | ./track_meters.py
trap "kill $RTL_PID" EXIT
```

Make it executable:
```bash
chmod +x log_water_meter.sh
```

Run everything in one command:
```bash
./log_water_meter.sh
```

---

## Summary cheat sheet

| Command                        | Purpose                             |
|--------------------------------|-------------------------------------|
| `rtl_test -t`                   | Verify RTL-SDR hardware is working  |
| `rtl_tcp`                       | Start SDR stream server             |
| `./rtlamr -msgtype=r900`        | Decode Neptune R900 water meters    |
| `./track_meters.py`             | Label & log potable vs irrigation   |
| `./log_water_meter.sh`          | Run everything and log automatically|

---

## Congratulations!

You are now:
- Reading your Neptune water meter broadcasts live
- Tracking and labeling potable vs irrigation usage
- Logging everything for later graphs or billing review

---

## Want to add daily CSV summaries or graphs?

Open an issue or PR. We can extend this project to generate hourly or daily usage plots. 🚀
