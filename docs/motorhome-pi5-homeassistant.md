# Motorhome Pi5 — Home Assistant

## Status

> **Not yet implemented.** This document describes the planned hardware and intended purpose of this project. No software has been installed and no configuration has been applied. This document will be updated as implementation progresses.

---

## Purpose

This project extends the home server portfolio with a mobile-first automation platform. Where the Nextcloud Raspberry Pi serves as a home server for personal cloud storage, the Motorhome Pi5 is intended to run Home Assistant as a self-contained smart home and vehicle monitoring system installed in a motorhome.

The two projects share the same philosophy: self-hosted, low-power, personally owned infrastructure with no dependency on third-party cloud services.

---

## Hardware

| Component | Detail |
|---|---|
| Board | Raspberry Pi 5 |
| RAM | 8 GB |
| Use case | Mobile, vehicle-mounted |
| Target software | Home Assistant |

All other hardware details including storage, power supply, case, and networking are [TO BE DOCUMENTED] once the physical build begins.

---

## Intended Purpose

Home Assistant will be used to automate and monitor systems within the motorhome. Specific integrations, devices, and automations have not yet been defined and will be documented as the implementation takes shape.

Possible use areas include but are not limited to:

- Environmental monitoring inside the vehicle
- Power and battery management visibility
- Lighting or appliance control
- Mobile connectivity and remote access

> These are areas of interest, not confirmed implementations. Nothing listed above should be treated as a completed or planned feature until it appears in an updated version of this document with an implementation status.

---

## Relationship to Nextcloud Home Server

This project is a sibling to the [Nextcloud Home Server](docs/nextcloud-home-server.md). Both run on Raspberry Pi hardware and follow the same self-hosted ownership model. They are independent systems with different purposes and are not currently planned to integrate with each other.

---

## Implementation Log

| Date | Milestone |
|---|---|
| [TO BE DOCUMENTED] | Hardware acquired |
| [TO BE DOCUMENTED] | Home Assistant installed |
| [TO BE DOCUMENTED] | First automation configured |

---

## Next Steps

The following steps are anticipated before this project moves from planned to active status.

1. Acquire and assemble hardware.
2. Select Home Assistant installation method (Home Assistant OS, Supervised, or Container).
3. Document power supply and network configuration for vehicle environment.
4. Perform initial Home Assistant setup and record configuration decisions.
5. Update this document to reflect actual implementation state.

---

## Related Documents

- `docs/nextcloud-home-server.md`
