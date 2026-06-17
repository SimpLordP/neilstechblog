---
title: "The Homelab Hero Part 1: The Hardware"
date: 2026-06-16
draft: false
tags: ["Homelab", "Raspberry Pi", "Hardware", "Networking"]
series: ["Homelab Hero"]
summary: "Part 1 of building a 4-node Raspberry Pi K3s cluster. Hardware choices, rack build, and physical setup."
description: "Part 1 of building a 4-node Raspberry Pi K3s cluster in a 10-inch mini-rack. Starting with the hardware choices and physical build."
---

# How it began

My homelab obsession started the same way most do - a YouTube rabbit hole at 2am. I was watching
videos of engineers building compact Kubernetes clusters when I realized I had most of the hardware
sitting in a box already. Four Raspberry Pi 4s, a cheap unmanaged switch, and a pile of ethernet
cables.

What I didn't have was a plan. Or a rack. Or really any idea what I was getting into.

This is Part 1 of that journey - the hardware decisions, the physical build, and everything I
wish I'd known before I started zip-tying cables to a shelf bracket.

# The Goal

Before buying anything, I set a clear requirement: build a production-grade Kubernetes cluster
that runs 24/7, fits on a desk, costs under $500 total, and actually teaches me something about
how enterprise infrastructure works at a small scale.

The end state I was targeting:

- 1x Control Plane node
- 3x Worker nodes
- Clean cable management
- 10-inch mini-rack form factor
- Single power strip, no power brick chaos

# The Hardware

## Raspberry Pi 4

The obvious choice for a budget ARM cluster. I went with four Pi 4B units - one 4GB model for
the control plane and three 8GB models for the workers. The 8GB models matter for Kubernetes
worker nodes because pod scheduling needs headroom above what the OS consumes.

Each Pi runs off a dedicated USB-C power cable routed back to a single multi-port Anker power
brick mounted behind the rack.

## MicroSD Cards

256GB Samsung Pro Endurance cards. The "Pro Endurance" line matters here - standard MicroSD
cards are rated for consumer read/write cycles and will fail within months under continuous
Kubernetes etcd writes. The endurance series is designed for dashcam and surveillance workloads
which have a similar sustained write profile.

## Netgear GS308PP Switch

8-port unmanaged PoE+ switch. The PoE capability was originally planned to power the Pis
directly over ethernet, eliminating USB-C power cables entirely. In practice, the Pi 4's
USB-C power requirements don't play well with 802.3af PoE without a HAT, so I ended up using
it purely as a dumb switch. Still a solid piece of hardware - zero configuration, zero
maintenance.

## The 10-Inch Mini-Rack

The GeekPi 8U 10-inch server cabinet. Compact, stackable, and just large enough to fit four
Pi nodes, a switch, and a patch panel with room to spare. If I were doing this again I'd go
12-inch for the extra breathing room, but the 10-inch works.

# The Physical Build

Assembly order matters more than most people think. I learned this the hard way after having
to tear everything apart twice.

The order that actually works:

1. Mount the switch at the top
2. Run all ethernet patch cables before mounting nodes
3. Mount Pi nodes bottom to top
4. Route power cables last along the back rail
5. Label everything before closing up

Cable management on a 10-inch rack is an exercise in patience. Every centimeter counts.
I used flat ethernet cables and right-angle USB-C adapters to keep things tight.

# Current State

The physical rack is complete. All four nodes power on, get DHCP leases from the Linksys
router, and are reachable via SSH from my WSL terminal.

In Part 2 we'll cover OS provisioning - flashing Debian, configuring hostnames, disabling
WiFi, and getting the kernel cgroup parameters set correctly for Kubernetes.

# Hardware Links

- [GeekPi 8U 10-inch Mini Rack](https://www.amazon.com/s?k=geekpi+10+inch+mini+rack)
- [Raspberry Pi 4B 8GB](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
- [Samsung Pro Endurance 256GB MicroSD](https://www.amazon.com/s?k=samsung+pro+endurance+256gb)
- [Netgear GS308PP PoE+ Switch](https://www.amazon.com/s?k=netgear+gs308pp)
- [Anker Multi-Port USB-C Power Strip](https://www.amazon.com/s?k=anker+usb+c+power+strip)
