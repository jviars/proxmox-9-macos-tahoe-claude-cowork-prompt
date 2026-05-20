Proxmox 9 macOS Tahoe VM Provisioning Asset
This repository contains a high-density, one-shot provisioning blueprint engineered to cleanly spin up a macOS Tahoe (26) virtual machine inside a Proxmox VE 9 hypervisor environment without requiring native Apple hardware.

🎯 What This Is For
This asset is specifically formatted to serve as a context layer for AI Agents and Coworkers (such as Claude). It allows an AI assistant to understand the exact hypervisor flags, kernel tweaks, and structural quirks required to successfully build and patch the VM on modern Proxmox hosts without failing compliance or validation checks.

⚠️ User Notice
You are completely welcome to utilize the setup instructions to configure the virtual machine yourself if you would like! However, please keep in mind that this configuration path is not optimized to be "user-friendly."

It relies heavily on command-line storage manipulation, raw disk image imports via qm importdisk, and manual Python config overrides rather than a standard GUI wizard experience.
