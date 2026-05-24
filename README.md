# ⚡ Stop Writing YAML for Modbus: Bridge RS485 Inverters to Home Assistant via MQTT & JSON

If you run a smart home, you’ve probably hit this wall: You have a solar inverter (Growatt, Deye), a smart energy meter (Eastron SDM120), or an HVAC system. They hold a goldmine of data, but they speak a 40-year-old industrial language: **RS485 Modbus RTU**. Home Assistant and Node-RED, on the other hand, love **MQTT and JSON**. 

The traditional DIY approach is to buy a cheap USB-to-RS485 dongle, plug it into your Raspberry Pi, and spend your weekend writing complex Modbus polling logic in YAML. Then you deal with byte-swapping errors, dropping connections, and RS485 bus collisions.

**The Goal of this Guide:** We bypass the coding completely. This walkthrough shows how to use an intelligent Edge Gateway to automatically poll a Modbus RTU device, package the raw hex data into clean JSON, and publish it natively to your Mosquitto MQTT broker.

---

## 1. The Problem: What Raw Modbus Actually Looks Like
Let's say we want to read the current "Active Power". The meter expects a specific hexadecimal query on the RS485 line. When queried, it spits back raw binary data. If the power is `2500 Watts`, the raw response looks like this:

`01 04 04 45 1C 40 00 3A 5F`

* `01` = Slave ID
* `45 1C 40 00` = The actual payload (IEEE 754 Floating Point for 2500.0)

Home Assistant has absolutely no idea what to do with `45 1C 40 00`. You normally have to write data templates to decode 32-bit floats. It’s a headache.

## 2. The Hardware Bridge (The "No-Code" Approach)
Instead of forcing HA to be a SCADA controller, we delegate the translation to a dedicated hardware bridge. For this setup, we use the **VALTORIS VT-DTU500** series. 

* **Storage Modbus (Zero-Collision):** The gateway autonomously polls the meter every second and caches the data in its RAM. 
* **Native JSON Parsing:** It strips away the CRC, decodes the 32-bit floats, and formats it into a human-readable JSON string.
* **Direct MQTT Uplink:** It acts as an MQTT client, publishing directly to your local Mosquitto broker.

## 3. The Configuration That Matters
Log into the gateway's web interface. Here are the critical parameters:

| Parameter | Setting Example | Why it matters |
| :--- | :--- | :--- |
| **Baud Rate** | `9600, 8, N, 1` | Must match your Solar Inverter. |
| **Transfer Protocol** | `Modbus RTU to JSON` | Tells the gateway to decode the hex. |
| **Byte Order** | `CDAB` or `ABCD` | Crucial for 32-bit floats. Get this wrong, and 2500W becomes 0.00003W. |
| **MQTT Broker IP**| `192.168.1.100` | Points to your HA Mosquitto add-on. |
| **Publish Topic** | `home/solar/power` | The topic HA will subscribe to. |

## 4. What Home Assistant Actually Sees
You don't need to write a single line of Modbus polling code. The gateway automatically publishes clean payloads like this:

```json
{
  "device_id": "VALTORIS_DTU",
  "slave_id": 1,
  "data": {
    "active_power_W": 2500.0,
    "total_energy_kWh": 1450.5
  }
}
```

## 5. Adding it to Home Assistant
Adding it to your HA dashboard is trivial. Just copy the `mqtt_sensor.yaml` provided in this repository into your HA configuration.

---
*For the exact wiring diagrams and hardware specs of the Edge Gateway used in this integration, visit the [VALTORIS VT-DTU500 Official Page](https://valtoris.com/product-center/industrial-cellular-modem/).*
