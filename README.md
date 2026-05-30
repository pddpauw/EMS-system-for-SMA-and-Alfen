# EMS-system-for-SMA-and-Alfen
full EMS system to control Inverter, EV charger and Home Battery and using electricity pricing

# DIY Energy Management System (EMS) for SMA + Battery + EV Charging using Node-RED

## Hardware Setup

The system is built around the following components:

### Solar Installation

* SMA Sunny Tripower inverter
* SMA Sunny Home Manager 2.0

### Battery Storage

* SMA Sunny Boy Storage SBS 5.0
* High-voltage battery connected to the SBS

### EV Charging

* Alfen EV charger
* Modbus TCP enabled

### Control Platform

* Node-RED
* Modbus TCP communication with all devices
* Custom dashboards for monitoring and control

---

# Project Objective

The goal was to build a fully automated Energy Management System (EMS) that optimizes:

1. Self-consumption of solar energy
2. Battery charging and discharging
3. EV charging
4. Dynamic electricity pricing
5. Negative injection price situations

while maintaining full transparency through a custom dashboard and diagnostics system.

The EMS continuously balances:

* Solar production
* Household consumption
* Battery charging/discharging
* Grid import/export
* EV charging demand
* Electricity purchase prices
* Electricity injection prices

---

# High-Level Decision Framework

The EMS continuously answers four questions:

### 1. Should solar production be limited?

Depends on:

* Current injection price
* Household consumption
* Battery state of charge
* EV charging demand

### 2. Should the battery charge or discharge?

Depends on:

* State of charge
* Electricity price
* Available solar production
* Current export/import

### 3. Should the EV charge?

Depends on:

* Vehicle connected status
* User charging permission
* Solar availability
* Active charging schedules
* Selected charging mode

### 4. How much power should be exchanged with the grid?

The preferred order is:

1. Use solar locally
2. Charge the battery
3. Charge the EV
4. Export to the grid

Only when economically beneficial.

---

# Data Sources

## Dynamic Electricity Prices

Prices are downloaded from Energy-Charts.

Three datasets are stored:

* Yesterday
* Today
* Tomorrow

Both consumption and injection prices are processed and stored as quarter-hour arrays.

These prices drive:

* Battery strategy
* Solar limiting strategy
* EV charging decisions
* Winter mode operation

---

# Solar Control Logic

## Solar Auto Mode

The inverter output is automatically controlled by the EMS.

### Positive Injection Prices

When export still has value:

* Solar production is allowed to run freely
* The inverter limit is set to 100%

### Negative Injection Prices

When exporting electricity costs money:

The EMS attempts to:

* Minimize exports
* Continue supplying household loads
* Continue charging the battery if beneficial

A minimum solar floor is maintained:

```text
Household consumption + safety margin
```

to avoid importing electricity unnecessarily.

---

# Battery Control Logic

## Battery Auto Mode

The battery is controlled through Modbus registers.

### Charging

Battery charging is prioritized when:

* Excess solar is available
* Electricity prices are negative
* State of charge is below target

### Discharging

Battery discharge is allowed when:

* Electricity prices are high
* Household consumption exists
* It is economically beneficial

### Control Registers

The EMS writes:

* Register 40151 → External battery control enable
* Register 40149 → Charge/discharge power setpoint

Writes are rate-limited to avoid unnecessary Modbus traffic.

---

# EV Charging Logic

## Master Enable Switch

The charging system includes a master switch:

### Enable Charging = OFF

The EV never charges.

This overrides:

* Manual charging
* Solar charging
* Timed charging

### Enable Charging = ON

Charging is allowed according to the selected strategy.

---

## Solar EV Charging

When:

* Vehicle connected
* Enable Charging enabled
* Solar EV mode enabled

the EMS calculates available solar capacity and dynamically adjusts the EV charging current.

The goal is to absorb solar surplus while minimizing grid import.

---

## EV Minimum Charging Threshold

A practical minimum charging power of approximately:

```text
4 kW
```

is used.

Although the Alfen charger can technically accept lower currents, many vehicles terminate charging sessions below approximately 6A per phase.

---

# Solar Availability Challenge

One important discovery during development:

The SMA inverter only reports actual production.

It does **not** report:

```text
Potential production if output limits were removed
```

This means that when the inverter is already limited, the EMS cannot directly determine how much unused solar is available.

---

# Solar Probe Strategy

To solve this limitation, a solar probing mechanism was introduced.

When:

* Battery is nearly full
* EV is connected
* Solar charging is active

the EMS periodically:

1. Temporarily removes solar limitations
2. Measures the resulting export increase
3. Estimates available solar capacity
4. Adjusts EV charging accordingly

This allows the EMS to estimate unused solar potential without additional hardware.

---

# Operating Modes

## Summer Mode

Primary objective:

* Maximize self-consumption
* Minimize exports during negative injection pricing
* Charge EV from solar whenever possible

---

## Winter Mode

Primary objective:

* Exploit cheap grid energy
* Charge battery during low-price periods
* Support scheduled EV charging

User-configurable thresholds determine charging behavior.

---

## SMA Mode

Manual mode.

The EMS does not actively manage energy flows.

All control remains with the SMA ecosystem.

---

# Dashboard Features

A custom Node-RED dashboard was developed including:

## EMS Control

* Summer mode
* Winter mode
* SMA mode

## Battery Control

* Auto mode
* Hold mode
* Charge mode

## Solar Control

* Auto mode
* Manual output limiting

## EV Charging

* Enable charging
* Manual charging
* Solar charging
* Scheduled charging

---

# EV Charging Timer System

A complete scheduling system was implemented.

Users can define:

* Charging days
* Start times
* Charging duration
* Charging power

This system is similar to the scheduling functionality used elsewhere in the Node-RED environment.

---

# Diagnostics & Monitoring

A dedicated diagnostics dashboard was added.

The system continuously monitors:

| Device      | Read Status | Write Status |
| ----------- | ----------- | ------------ |
| SMA PV      | OK          | OK           |
| SMA Battery | OK          | OK           |
| Alfen EV    | OK          | OK           |

Green indicators show healthy communication.

Red indicators highlight communication failures or missing updates.

---

# Verification Registers

To validate that commands are actually accepted, verification reads were introduced.

### SMA PV

Reads the active power limit register and compares it with the requested value.

### SMA Battery

Reads the active battery setpoint and compares it with the requested value.

### Alfen EV

Reads the actual applied charging current and compares it with the requested charging current.

This provides significantly more confidence than merely checking whether a Modbus write was sent.

---

# Modbus Strategy

## SMA Battery

Cyclic writes to the battery control registers are supported and considered safe.

---

## SMA PV

Frequent updates to the active power limit register are standard EMS practice.

---

## Alfen EV Charger

Periodic refreshes of charging current commands are required because the charger includes a command timeout mechanism.

The EMS refreshes charging commands approximately every 30 seconds.

---

# Key Lessons Learned

### Negative injection pricing changes the optimization target

The objective becomes:

```text
Minimize exports
```

not:

```text
Minimize solar production
```

These are fundamentally different strategies.

---

### Battery SoC alone is not enough

A battery at 99% SoC may already restrict charging due to internal management and balancing.

---

### Potential solar production cannot be directly measured

Additional irradiance sensors or production probes are required if precise solar potential estimation is desired.

---

### Verification is more important than command transmission

A successful Modbus write does not necessarily mean a device accepted or applied the requested value.

Verification registers provide much greater reliability.

---

# Current Result

The resulting EMS provides:

* Dynamic price optimization
* Automated battery charging and discharging
* Solar production management
* Solar-aware EV charging
* Timed EV charging
* Modbus diagnostics
* Communication health monitoring
* Command verification
* Full dashboard control

The entire system is implemented in Node-RED using Modbus TCP and runs fully autonomously while remaining transparent and user-configurable.

