# SBOP System Overview

**Document ID:** ARC-SYS-001
**Version:** 2.0
**Status:** Approved
**Last Review:** 2026-04-28
**Owner:** System Architecture Team

---

## 1. Purpose

The SBOP (Secure Boot & OTA Platform) defines a hardware-agnostic security framework for ensuring firmware authenticity, integrity, and controlled deployment across embedded devices.

This document provides a high-level overview of system scope, objectives, and architectural boundaries.

---

## 2. System Objectives

The system is designed to achieve the following:

* Ensure only authenticated firmware is executed on devices
* Provide secure and reliable OTA update mechanisms
* Establish device identity and cryptographic trust anchors
* Prevent unauthorized cloning and rollback attacks
* Support scalable backend-driven device lifecycle management

---

## 3. System Scope

SBOP includes:

* Device-side secure boot and update logic
* Backend infrastructure for firmware distribution and key management
* Provisioning and manufacturing integration
* Security enforcement across firmware lifecycle

SBOP excludes:

* Application-level business logic
* Hardware-specific driver implementations
* Physical security mechanisms outside defined threat model

---

## 4. System Components

The system is composed of the following logical components:

* Boot Component (Secure Boot Execution)
* Update Component (OTA Management)
* Identity Component (Device Identity & Provisioning)
* Crypto Component (Key and Cryptographic Operations)
* Backend Component (Control Plane)
* Toolchain Component (Build, Signing, Deployment)

---

## 5. Trust Boundaries

The system defines the following trust boundaries:

* Device Boundary
* Backend Boundary
* Manufacturing Boundary

Each boundary enforces distinct security controls and trust assumptions.

---

## 6. Assumptions

* Devices have access to a unique hardware identifier or equivalent entropy source
* Cryptographic primitives are correctly implemented or provided
* Secure storage is available for root keys or key derivation

---

## 7. Non-Goals

* No guarantee against invasive physical attacks beyond defined threat model
* No dependency on specific MCU or hardware platform
* No runtime protection of application-layer vulnerabilities

---

## 8. Document References

* Requirements Specification
* System Design Specification
* Security Model
* Verification Plan
