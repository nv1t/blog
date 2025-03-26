---
title: Treadmill Telemetry - Real-Time BLE Metrics
date: 2025-03-26
tags:
- 'reverse engineering'
- 'treadmill'
published: true
slug: treadmill-telemetry
layout: post
images:
- '/blog/img/2025/PXL_20250326_094740373.jpg'
---

A while ago, I picked up a **Sportstech S-Walk treadmill** — a compact walking pad with built-in Bluetooth support. Naturally, I wanted to get _more_ out of it than the default app experience and the default app just sucked big time. So I started digging into the treadmill’s **BLE (Bluetooth Low Energy)** capabilities to read real-time speed, distance, and workout time. I found myself in **FTMS** (Fitness Machine Service), a standard for communicating protocol with fitness equipment over BLE.

This post walks through the process of:
- Connecting to the treadmill over BLE
- Understanding the data sent from the treadmill
- Logging live workout data with Python
- Aggregating and analyzing the workouts

It’s all done in Python, working directly with Bluetooth Low Energy through `bleak` to read the data.

# First Contact: BLE Connection Basics

At the heart of **Bluetooth Low Energy (BLE)** communication lies the **Generic Attribute Profile (GATT)** — a hierarchical data model built around **services** and **characteristics**.

A **characteristic** represents a **typed, addressable data value** within a service. Each characteristic is defined by:

- A **UUID**: A unique 128-bit or 16-bit identifier (e.g., `0x2ACD` for Treadmill Data)
- A **value**: The actual data payload (e.g., speed, distance, etc.)
- A set of **properties**: Permissions and behaviors like:
    - `Read` – client can request the current value
    - `Write` – client can send a value to the device
    - `Notify` – device can push value changes to the client
    - `Indicate` – like Notify, but with acknowledgment

Characteristics are where most real interaction with a BLE device occurs. For example, the **Fitness Machine Service (FTMS)**, standardized by the Bluetooth SIG, exposes several characteristics that map to specific telemetry or control points on fitness machines.

In most fitness scenarios, clients (like our Python script) **subscribe to `Notify` characteristics** to receive real-time updates. Instead of polling the device repeatedly, the device pushes updates any time the value changes — such as every second during a workout.

# BLE Data: Who Are You and What Can You Do?_
After successfully connecting to the device, I query several key **GATT characteristics** to retrieve information about the treadmill's identity and capabilities. These values are part of the standard **Device Information Service** and **FTMS (Fitness Machine Service)**.

(Source: https://gist.github.com/nv1t/3856d1f23c7d7696bdaf6ba0d4d1c17b#file-swalk-read-py)
```python
async def main(addr):
	# [...] unimportant Source Code redacted

    # Find the BLE device by address
    device = await BleakScanner.find_device_by_address(
        addr, cb=dict(use_bdaddr=False)
    )
    if device is None:
        logger.error("could not find device with address '69:82:20:D9:27:90'")
        return

    logger.info("connecting to device...")

    # Connect to the device
    async with BleakClient(device) as client:
        logger.info("Connected")

        # Read device characteristics
        manufacturer = await client.read_gatt_char("00002a29-0000-1000-8000-00805f9b34fb")
        model = await client.read_gatt_char("00002a24-0000-1000-8000-00805f9b34fb")
        hardware_revision = await client.read_gatt_char("00002a27-0000-1000-8000-00805f9b34fb")
        firmware_revision = await client.read_gatt_char("00002a26-0000-1000-8000-00805f9b34fb")

        fitness_machine_feature = await client.read_gatt_char("00002acc-0000-1000-8000-00805f9b34fb")
        supported_speed_range = await client.read_gatt_char("00002ad4-0000-1000-8000-00805f9b34fb")

        # Log the device characteristics
        logger.info("Manufacturer: %s", str(manufacturer, "utf-8"))
        logger.info("Model: %s", str(model, "utf-8"))
        logger.info("Hardware Revision: %s", str(hardware_revision, "utf-8"))
        logger.info("Firmware Revision: %s", str(firmware_revision, 'utf-8'))

        logger.info("Supported Speed Range: %s", supported_speed_range.hex())

        logger.info("Fitness Machine Feature: %s", fitness_machine_feature.hex())

        # Start notification for a specific characteristic and handle notifications in a separate function
        await client.start_notify("00002acd-0000-1000-8000-00805f9b34fb", notification_handler)

        # Main loop to keep the program running until stopped
        while not stopped:
            await asyncio.sleep(1)

        # Stop notification for a specific characteristic
        await client.stop_notify("00002acd-0000-1000-8000-00805f9b34fb")
```

Here's what a typical connection log looks like:

```
-> % python read.py
2024-02-07 16:32:49,638 __main__ INFO: starting scan...
2024-02-07 16:32:58,795 __main__ INFO: connecting to device...
2024-02-07 16:32:59,838 __main__ INFO: Connected
2024-02-07 16:33:00,505 __main__ INFO: Manufacturer: FITHOME
2024-02-07 16:33:00,505 __main__ INFO: Model: JJ-BT2-DL
2024-02-07 16:33:00,506 __main__ INFO: Hardware Revision: 1.0
2024-02-07 16:33:00,506 __main__ INFO: Firmware Revision: 1.017
2024-02-07 16:33:00,506 __main__ INFO: Supported Speed Range: 640058020a00
2024-02-07 16:33:00,506 __main__ INFO: Fitness Machine Feature: 0416000001000000
```

The values returned by these characteristics reveal quite a bit about the device. The **Manufacturer** and **Model** identifiers confirm that it's a FITHOME treadmill, specifically the `JJ-BT2-DL` model. The **Firmware** and **Hardware Revisions** help track the software version (`1.017`) and hardware iteration (`1.0`) I’m working with. The **Supported Speed Range** provides insight into the treadmill’s operational limits, while the **Fitness Machine Feature** bitfield indicates which FTMS capabilities the device supports — such as target speed control or real-time metrics.

# Decoding BLE Capabilities
The treadmill advertises its **capabilities and supported ranges** through two key GATT characteristics: `Supported Speed Range` and `Fitness Machine Feature`. These are encoded as raw byte strings, but with a little decoding, they reveal exactly what the machine can (and can’t) do.

## Supported Speed Range
When I queried the treadmill, I got the following:
```
2024-02-07 16:33:00,506 __main__ INFO: Supported Speed Range: 640058020a00
```

This 6-byte payload encodes three unsigned 16-bit integers (in little-endian order):
- `64 00` → `0x0064` → **100** → **1.00 km/h** (min speed)
- `58 02` → `0x0258` → **600** → **6.00 km/h** (max speed)
- `0a 00` → `0x000A` → **10** → **0.10 km/h** (step size)

So the treadmill supports target speeds between **1.0 and 6.0 km/h**, adjustable in **0.1 km/h** increments.

## Fitness Machine Feature

The `Fitness Machine Feature` field defines what optional FTMS features the treadmill supports. My device reported:

```
2024-02-07 16:33:00,506 __main__ INFO: Fitness Machine Feature: 0416000001000000
```

This is an 8-byte bitfield, split into two 4-byte segments:
- `04160000` → Treadmill-specific features
- `01000000` → General machine features

Breaking it down (bitwise), this treadmill advertises support for:
- **Speed Range** control (bit 2)
- **Heart Rate Measurement** (bit 10)
- **Total Distance Reporting** (bit 13)
- **Incline Control** (bit 14)
- **User Data Support** (bit 0 of general features)

Some of these features may require external accessories (e.g., a heart rate monitor), but it’s good to know they’re recognized at the protocol level.

## Live Metrics

To access the treadmill’s live workout metrics — like speed, distance, and elapsed time — I needed to **subscribe to a specific GATT characteristic** that supports notifications. In this case, the characteristic with UUID `0x2ACD` (defined by the FTMS spec as "Treadmill Data") is responsible for streaming real-time performance data. By enabling notifications on this characteristic, the treadmill automatically pushes updates to my script whenever new data is available, eliminating the need for constant polling.

```python
await client.start_notify("00002acd-0000-1000-8000-00805f9b34fb", notification_handler)
```

### From Hex to Human
Once subscribed, the device continuously pushes data to my script in the form of a **hexadecimal byte string**, which is then parsed by the function `notification_handler()`

```python
def notification_handler(characteristic: BleakGATTCharacteristic, data: bytearray):
	"""
	Handle notifications by logging the received data and appending it to a CSV file.

	Args:
		characteristic (BleakGATTCharacteristic): The characteristic the data is received from.
		data (bytearray): The data received.
	Returns:
		None
	"""
	# Log the received data
	logger.info("%s: %r", characteristic.description, data.hex())

	# Extract speed, distance, and timedelta from the data
	speed = struct.unpack("h", data[2:4])[0]
	distance = struct.unpack("h", data[4:6])[0]
	timedelta = struct.unpack("h", data[13:15])[0]

	# Check if the log file for the current date exists
	current_date = datetime.datetime.now().strftime("%Y-%m-%d")
	file_name = f"logs/{current_date}.log.csv"
	if not os.path.exists(file_name):
		with open(file_name, "w") as f:
			f.write("Workout (UUID), Speed (n/100 km/h), Distance (meter), Time (seconds)")
			f.write("\n")

	# Append the data to the log file
	with open(file_name, "a") as f:
		f.write("%s,%d,%d,%d,%d" % (str(workout_id),int(time.time()), speed, distance, timedelta))
		f.write("\n")
```

The payload received looks something like this:

```
2024-02-07 16:33:00,594 __main__ INFO: Treadmill Data: '840500000000000000000000000000'
```

Each notification contains a compact payload encoding values like speed, distance, and time. My script parses this raw data using `struct.unpack`, extracting the relevant values and converting them into usable variables for logging and analysis.

![A color-coded data chart displays a sequence of blocks, each containing two-character values. Labels point to specific segments: "Speed" (red, values "a0", "00"), "Distance" (green, "53", "00"), "Calories?" (yellow, "00", "03"), and "Time" (purple, "ca", "00"). Remaining blocks are mostly light blue with values like "84", "05", and "00".](/img/2025/2fb42fca7948c6d3a5782767f4fc748d.png)

```python
speed = struct.unpack("h", data[2:4])[0] # h => 2 byte signed integer
distance = struct.unpack("h", data[4:6])[0]
timedelta = struct.unpack("h", data[13:15])[0]
```

Afterwards this is logged into a CSV file for later retrieval and aggregation in the form of:

```
Workout (UUID), Timestamp, Speed (n/100 km/h), Distance (m), Time (s)
8a21..., 1708532560, 115, 32, 47
8a21..., 1708532561, 117, 34, 48
```

# Turning Raw Logs into Clean Data

To make the data easier to import into platforms like **Garmin**, **Strava**, or other fitness tracking tools, I wrote a simple aggregation script. It reads through all the individual daily CSV log files, extracts the relevant workout entries, and combines them into a single, clean summary. This aggregated output includes total distance, total workout time, and timestamps — making it much more convenient to convert into GPX, TCX, or other formats these platforms understand.

![Workout summary showing final workout data and combined workout data. Final workout has ID dcd51360-f8c9-4169-8a02-40310eb16abb, with a distance of 4 meters and time of 16 seconds. Combined workout data starts at 2025-03-26 10:47:04, with total distance of 0.00 km and total time of 0 hours 0 minutes 16 seconds.](/img/2025/2025-03-26_10-49.png)

(Source: https://gist.github.com/nv1t/3856d1f23c7d7696bdaf6ba0d4d1c17b#file-aggregate-py)
```python
starting_timestamp = None
workout_data = {}
# Check if the log file for the current date exists
current_dir = os.path.dirname(os.path.abspath(__file__))
current_date = datetime.now().strftime("%Y-%m-%d")
file_name = f"{current_dir}/logs/{current_date}.log.csv"
if not os.path.exists(file_name):
    sys.exit()


with open(file_name) as csvfile:
    reader = csv.reader(csvfile)
    next(reader)
    # Initialize an empty dictionary to store final results
    workout_data = {}

    for workout, timestamp, speed, distance, time in reader:
        if starting_timestamp is None:
            starting_timestamp = int(timestamp)

        # Check if workout exists in the dictionary
        if workout not in workout_data:
            workout_data[workout] = {"distance": 0, "time": 0}

        # Update distance and time if current values are higher
        if int(distance) > workout_data[workout]["distance"]:
            workout_data[workout]["distance"] = int(distance)
        if int(time) > workout_data[workout]["time"]:
            workout_data[workout]["time"] = int(time)

print("Final Workout Data:")
for workout, data in workout_data.items():
    print(f"\tWorkout: {workout}")
    print(f"\t\tDistance: {data['distance']} meters")
    print(f"\t\tTime: {data['time']} seconds")
```

This is one of those 'works on my machine' solutions—feel free to clean it up!

# Where to Go From Here?
- **Automatically convert this treadmill data into a GPX or TCX file to upload to Strava!**
    _(And what tools or libraries would make that easiest?)_
- **Is it possible to send commands back to the treadmill (e.g., start/stop, change speed) using the FTMS Control Point?**
    _(How safe is it to try?)_
- **How standardized is FTMS across different treadmill brands?**
    _(Could this script work for other models with little or no changes?)_


# Resources
- Parsing Fitness Machine Data in GATT: https://jjmtaylor.com/post/fitness-machine-service-ftms/
- Reversing a Treadmill: https://taylorbowland.com/posts/treadmill-wrapping-up/
- pycling ftms implementation of indoor trainers: https://github.com/zacharyedwardbull/pycycling
- FTMS Documentation: https://www.bluetooth.org/DocMan/handlers/DownloadDoc.ashx?doc_id=423422
- Source Code: https://gist.github.com/nv1t/3856d1f23c7d7696bdaf6ba0d4d1c17b%
