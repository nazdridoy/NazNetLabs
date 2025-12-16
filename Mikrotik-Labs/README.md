# MikroTik Labs

This directory contains configuration scripts and lab scenarios for MikroTik RouterOS devices.

## Available Labs

### 1. Bandwidth Management & Web Filtering
- **[PCQ Bandwidth Management and Time-Based Web Filtering](./PCQ-Bandwidth-Management-and-Time-Based-Web-Filtering.md)**
    - **Features**: 
        - PCQ (Per Connection Queue) for fair bandwidth distribution.
        - Queue Tree for hierarchical bandwidth management.
        - Time-based blocking of Facebook and YouTube (9 AM - 5 PM).
        - DNS-based filtering using Regex and Static entries.
        - VLAN-ready interface structure.
    - **Topology**: Hub-and-spoke with 4 departments (Top-MGT, MKT, HR, FIN).
