# MOS SR-IOV i915 Driver

mos-sriov-driver provides a **MOS plugin** for managing the SR-IOV i915 driver.

---

## Overview

This repository contains the **MOS plugin implementation**, optional helper
functions, and configuration files (such as `settings.json`) required to
integrate SR-IOV driver management into MOS.

The plugin enables MOS to download and install SR-IOV drivers on demand directly
onto the system.

### Driver Source

- SR-IOV i915: [https://github.com/strongtz/i915-sriov-dkms](https://github.com/strongtz/i915-sriov-dkms)

No driver binaries are included in this repository.

---

## Build & Automation

This repository includes a **GitHub Actions workflow** used to build and package
the plugin for MOS.

The build process is fully automated and produces artifacts that can be consumed
by the MOS Hub or MOS release tooling.

---

## Licensing

The contents of this repository (plugin code, scripts, and configuration)
are licensed under **GPL-3.0**.

Downloaded SR-IOV i915 drivers remain licensed under their respective upstream licenses
and are not part of this repository.
