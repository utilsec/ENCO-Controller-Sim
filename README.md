# ENCO-SIM: Apartment Hydronic Heating SCADA Simulator

A vibe-coded ICS training simulator that emulates a small apartment-building hydronic heating system, complete with a Modbus/TCP slave, Telnet management interface, and a web-based HMI. Built for OT/ICS cybersecurity exercises — red teamers get realistic protocols to attack, blue teamers get a working physics model and defensive monitoring to detect the attacks.

Built by [Mike Holcomb](https://mikeholcomb.com). More OT/ICS tools and resources at [mikeholcomb.com](https://mikeholcomb.com).

## Why this exists

Most ICS lab targets are static — write a value, the value stays written, declare victory. That's not how real plants work. ENCO-SIM runs a continuous physics simulation underneath the protocol stack, so:

- Spoofed sensor values get overwritten every 2 seconds by the real thermal model. Sensor injection has to be **sustained**, not fire-and-forget.
- Coil writes get re-asserted by the control loop every second. Forcing a boiler off requires holding the coil down, not flipping it once.
- The control loop logs every detected mismatch between commanded and observed coil state. Blue team has something real to alert on.

The result is an environment where the difference between "I touched a register" and "I actually moved the process" matters — which is the whole point of OT security.

## Simulated process

A three-zone hydronic (hot water) heating system for a small apartment building:

- **Boiler** with ignition coil and pressure/flow sensors
- **Circulation pump** moving heated water through the loop
- **Three zone valves**, one per floor, gated by per-zone thermostat calls
- **Three 1-Wire zone temperature sensors** plus an outdoor ambient probe
- **Hysteretic thermostat control** with a 20.0 °C setpoint and ±0.5 °C deadband
- **Heat-loss model** tied to indoor/outdoor differential, so colder outdoor temps drive longer burn cycles

## Services and ports

| Service    | Default port | Purpose                                    |
|------------|--------------|--------------------------------------------|
| Modbus/TCP | 502          | Process I/O — coils, DIs, holding regs     |
| Telnet     | 23           | Management/console interface (configurable via CLI arg) |
| Web HMI    | 55555        | Operator view + manual coil overrides      |

## Requirements

- Python 3.9+
- `pymodbus` 3.x

```bash
pip install "pymodbus>=3.0,<4.0"
```

Note: ports 23 and 502 are privileged on Linux. Either run with `sudo`, grant the binary `cap_net_bind_service`, or override the Telnet port via the first CLI argument.

## Running it

```bash
# Default ports (Telnet 23, Modbus 502, HMI 55555)
sudo python3 enco_sim.py

# Telnet on a non-privileged port
python3 enco_sim.py 2323
```

Then point a browser at `http://<host>:55555/` for the HMI, or hit Modbus/TCP on `<host>:502` with your tool of choice (`mbtget`, `pymodbus.console`, Metasploit `modbusclient`, etc.).

## Modbus memory map

Indexes are zero-based as exposed on the wire (`zero_mode=True`).

### Holding Registers (FC 3 / FC 6 / FC 16)

| Address | Field                  | Encoding                       |
|---------|------------------------|--------------------------------|
| HR 0    | Zone 1 temperature     | int16, °C × 100                |
| HR 1    | Zone 2 temperature     | int16, °C × 100                |
| HR 2    | Zone 3 temperature     | int16, °C × 100                |
| HR 8    | Boiler pressure        | uint16, bar × 1000             |
| HR 9    | System flow rate       | uint16, L/min × 1000           |
| HR 10   | Outdoor ambient temp   | int16, °C × 1000               |

### Coils (FC 1 / FC 5 / FC 15)

| Address | Field                |
|---------|----------------------|
| CO 0    | Boiler ignition      |
| CO 1    | Circulation pump     |
| CO 2    | Zone 1 valve         |
| CO 3    | Zone 2 valve         |
| CO 4    | Zone 3 valve         |

### Discrete Inputs (FC 2)

| Address | Field                       |
|---------|-----------------------------|
| DI 1    | Boiler fault                |
| DI 2    | Low water cutoff            |
| DI 3    | Zone 1 call for heat        |
| DI 6    | Zone 2 call for heat        |

## Architecture

Two cooperating loops drive the simulation:

**Physics loop** (every 2 s)
Reads the commanded actuator state, advances the real thermal model for each zone (heat in from boiler+pump+valve, heat loss to outdoors), and publishes the resulting temperatures to the holding registers. The internal `real_zone_temps` list is the single source of truth — anything an attacker writes to HR 0/1/2 lives at most one tick before it gets overwritten.

**Control loop** (every 1 s)
Reads the holding registers as the controller's "view" of the world, applies hysteretic thermostat logic to update the call-for-heat discrete inputs, and drives the actuator coils accordingly. It also remembers the last value it commanded to each coil and logs a warning whenever the observed coil state has drifted from the commanded state between cycles.

## Defensive features (for blue-team exercises)

The simulator includes detection logic that students can tune, evade, or improve on:

- **Re-assertion of commanded outputs.** Every control cycle re-writes the coils to their commanded values. An attacker who flips a coil with FC 5 will see it flip back within a second.
- **Coil-drift logging.** When a coil's observed value differs from the last commanded value, the control loop emits a `WARNING` with the channel, observed value, and prior commanded value — a clean signal for log-based detection.
- **Physics-anchored sensor values.** Spoofing a temperature register gets overwritten on the next physics tick, forcing attackers into sustained-write patterns that show up as anomalous Modbus traffic.
- **First-cycle suppression.** The drift detector only fires once it has a prior commanded value, so it can't false-positive on startup.

## Suggested exercises

- **Sensor injection.** Hold HR 0 at a high value with a tight write loop. Watch the boiler shut off in the HMI; measure how the physics model recovers when the writes stop.
- **Forced shutdown.** Drive CO 0 (boiler) to 0 with FC 5 and observe the re-assertion behavior. Try to outpace the control loop.
- **Detection tuning.** Modify the control loop to push warnings to syslog/Splunk/your SIEM of choice and write a rule that alerts on coil drift.
- **HMI vs. ground truth.** Compare the HMI display (which reads from Modbus) against `real_zone_temps` during an attack to see how operator views can be deceived even when the process itself is unaffected.

## Caveats

This is a teaching simulator, not a production controller. It deliberately exposes Modbus and Telnet without authentication so students have something to attack. Run it on an isolated lab network or VM — never on a routable interface.

## License & attribution

Vibe coded by Mike Holcomb. Use it, fork it, break it, teach with it. Additional OT/ICS tools and resources at [mikeholcomb.com](https://mikeholcomb.com).
