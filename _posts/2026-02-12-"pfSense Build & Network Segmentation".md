---
title: "pfSense Build & Network Segmentation"
date: 2026-02-12 00:00:00 +/-TTTT
categories: [research, networking]
tags: [pfsense, hardware, router]     # TAG names should always be lowercase
image: "/assets/images/pfsense_rev.png"
---
## Executive Summary
This project explores what Network Segmentation means under the context of small/home office. It makes the argument for why segmentation can be important even in SoHo settings, especially with modern use cases and scenarios. This is then followed up with a custom router build showcasing off-the-shelf parts that should be accessible to most people. We then cover the installation and configuration of an enterprise-tier networking and firewall software called pfSense.

```
• PROJECT CONTENTS •
[1] Network Segmentation
[2] pfSense Router Build
[3] VLAN Configuration
[4] Additional Details
```

## Network Segmentation
### ♦️ Definition & Importance
Network Segmentation is a term often used in the cybersecurity and networking world to describe the practice of dividing computer networks into smaller, isolated segments (often called subnets) to allow for better control over network traffic, performance, and security.  
Often times, network segmentation is used for isolating high-value or security-sensitive devices from other common devices. An example of this in a home-office type setting may be a work computer that is used for conducting banking activites being put on a network that is separate from one serving the mobile devices of visiting guests.

In today's high-speed digital landscape, our devices help facilitate activities such as online banking and e-commerce. These activities often include the exchange of critical personal and financial information. These bits of information are still very much targeted by malicious entities and recent developments in the world of security have shown that safety is not guaranteed even in your own home network utilizing equipment you've bought and paid for.

Just like how we would usually carry out high-value transactions at a secure location in person, we should secure our home networks especially in the case of work-at-home type settings as they are becoming more and more ubiquitous. While network segmentation alone is not the be all end all to good home network security, the good news is that implementing a relatively simple yet highly effective measure such as network segmentation in a home or small office environment does not need to be complicated.

### ♦️ Implementation Methods
``Physical Segmentation``
- Involves creating separate networks by utilizing dedicated hardware such as switches and routers.
- Example: Work computer is connected to a dedicated LAN port on a smart switch instead of the home WiFi which shares its connection with many other devices.
 
``Software Defined Networks (SDN)``
- A software controller manages the entire network in realtime.
- Network is segmented at an architectural level.
- Widely used in data centers and cloud networks.

``Virtual Local Area Networks (VLAN)``
- Networks are segmented at a logical level.
- Operates at Layer 2 (Data Link).
- Assigns a unique tag to each packet/frame to create logical groupings.

### ♦️ Limitations
Typical networking equipment like routers supplied by ISPs are usually woefully inadequate for the purposes of safeguarding against modern day threats. The operating system is often stripped down to only enable simple functionality and maintenance. This could be done for a multitude of reasons like upselling customers to a more expensive subscription. Besides, the ISPs don't really want you to go messing about with the router settings in the first place anyway.

There are plenty of commercial solutions out there that can help you to achieve Network Segmentation quick and easy but they may also be costly and come with their own set of caveats. The ASUS router software exploits of recent times come to mind.

So, aside from upgrading your home router to a more premium, feature-full device, there is the option of building one from scratch. Networking devices like modems and routers are after all, essentially mini computers with their own set of processors, memory, and storage.

A DIY router does not need to consist of premium and/or specialized parts. In fact, spare parts from an old build or laptop can be repurposed in order to minimize cost.

### ♦️ The DIY Approach
> Benefits of building your own router to allow for effective segmentation.

These are some of the pros of going with a DIY approach:

``Complete Control``
- You decide what OS to run on the device and what settings are available.

``Reduced Cost of Maintenance``
- A failed router would usually necessitate a complete replacement due to it being a closed system that might even utilize proprietary design.
- A DIY router is essentially a small computer with individual parts that utilize standardized plugs, connectors, etc. that can be replaced as needed without having to discard the entire system.

``Customization``
- Often times, it is difficult to find an aftermarket router with the specs and features you need at the price point you may target.
- DIY routers allow full customization like being able to upgrade the processor or install multiple NICs.

Building your own router and pairing it with a robust software solution like OPNSense or pfSense for network segmentation is very much achieveable and not as difficult as it may sound. It just requires some research ― which this guide aims to simplify ― alongside decent understanding of networking concepts and how to configure network equipment.  

## pfSense Router Build
### ♦️ Hardware
Since we will be building a machine from scratch, this list of parts might look very similar to a typical DIY computer build used for gaming or rendering. A router is basically a small computer after all and building our own brings with it plenty of benefits such as relatively infinite levels of serviceability, potentially better performance and even security.

It should be stated that there is no need to purchase brand new parts. Used parts such as those that were used in this guide will do just fine ― and I even encourage it as it helps to reduce e-waste. Unless you're building for high availability (well outside the scope of this write up) a simple computer with a good Network Interface Card (NIC) should be plenty.

All in all, this build cost me a grand total of ~$40. And that's only because I had to buy new memory to replace a faulty stick. Otherwise the only cost would've been the NIC and SSD.

**Parts used:**
- Processor : ``Intel Core i3 6100 3.7GHz``
- Memory : ``Kingston HyperX 4GB 2400MHz DDR4``
- Motherboard : ``Gigabyte H110-M Gaming 3``
- Storage : ``Samsung PM871 120GB``
- Case & PSU : ``Slevcase Spotless w/ 300W Power Supply``
- Network Card : ``HP 332T powered by Broadcom BCM5720``
<img width="2028" height="1326" alt="Untitled" src="https://github.com/user-attachments/assets/79f9b67e-47d6-4fb5-8bec-12e59ba40b0a" />

### ♦️ Software
There are plenty of ways you can go about building a router. For example, you could run something like pfSense or OPNSense off of a Virtual Machine. You could even go with OpenWRT or Merlin (ASUS's own flavor of OpenWRT). But for this guide, we are going to be looking at a "native" setup of pfSense running on OpenBSD.

### ♦️ Why pfSense?<br>
As stated before, pfSense is open-source and is highly regarded in the world of security for its robust features and scalability. It also comes in a "Community Edition" that is free to download and use with no severe limitations, something you might come to expect from software this polished. Personally, I find the interface to be easy to navigate and user-friendly. It has also been absolutely reliable in the many months I've had it running.<br>

### ♦️ Where do I get pfSense?<br>
Typically, you would visit the pfSense.org site to download your preferred version (most likely the Community Edition) but the process of downloading seems a bit too intrusive with it requiring a registered account.
Most people usually use the following mirror link to obtain the exact version of pfSense for their use case eg. ``pfSense-CE-<"version_number">-RELEASE-amd64.iso.gz``<br>
LINK: https://atxfiles.netgate.com/mirror/downloads/

### ♦️ How To Install pfSense?<br>
If you've ever had to install an operating system, installing pfSense is more or less a similar experience if a little simpler. For the sake of brevity, this guide will only cover steps up to the point of getting pfSense functional. Configuring it and customizing settings to your home network's needs can be so vast that it could be its own guide.

### ♦️ Installation Steps
**STEP 1 - Download pfSense Install Image**
- Go to the link provided above to obtain an appropriate installation image of pfSense for your hardware.
- Choose the architecture and select a mirror (download link).

**STEP 2 - Prepare bootable media (USB Drive)**
- Ensure you have an empty USB Drive with space no less than 4GB.
- Use an imaging software like Rufus to write the pfSense image to the USB Drive.

**STEP 3 - Install pfSense**
- Connect a temporary display to your custom router.
- Boot from the USB Drive by selecting it in the BIOS or Boot Up menu.
- Follow the on-screen instructions.
- Select the disk you would like to install pfSense on.
- Choose the filesystem (ZFS is sufficient).
- Start the installation.

**STEP 4 - Initial Configuration**
- Go through the initial setup process.
- Assign the correct network device to the appropriate interface eg. set network card connected to modem or ISP PON device as WAN interface.
- Access pfSense interface by entering the pfSense IP address into a web browser.
- Login with the default credentials (admin, pfsense).
- Configure your network settings, firewall rules, and other desired settings.
<br><br>

## VLAN Configuration
### ♦️ VLANs
The following will be a simplified explanation of what VLANs are and how they can be utilized to implement Network Segmentation through pfSense. If you are looking for how to configure VLANs on a pfSense device, there is already excellent documentation available on the netgate website.<br>
LINK: https://docs.netgate.com/pfsense/en/latest/vlan/configuration.html

A Virtual Local Area Network or VLAN is a software approach to segmenting a network. It essentially divides a network into multiple domains by assigning and identifying a unique tag on data packets, more specifically Ethernet frames, passing through the Data Link Layer.

### ♦️ Network Segmentation via VLANs
The exact steps to creating and maintaining VLANs for network segmentation in pfSense are already outlined in the official documentation through the link above.

The process of segmenting networks with VLANs essentially involves creating logical domains for each device or set of devices based on needs like performance and security.

You would first plan the overall network structure. For example, VLAN 10 would be for the workstation computer whereas VLAN 20 would be for mobile devices, followed by VLAN 30 which is configured for visiting guests.

You would then define these VLANs by assigning a subnet and then configuring the devices to the specific VLANs as planned.

If utilizing multiple switches ― an advanced network setup ― you would then need to connect them together using "trunk links" to allow traffic from multiple VLANs to pass between the switches.

### ♦️ Best Practices
If your network environment necessitates a setup with numerous VLANs, it is absolutely recommended to keep a record of the IDs and assigned subnets in case there is a need to rebuild or reset the router. Most router software like pfSense offer a backup option for this.

You can also indirectly improve QoS for specific services like VoIP by segregating devices that utilize them frequently to their own VLAN.

Likewise, you should group highly sensitive devices like systems used for work or finance in their own VLANs with restrictive security settings.  


## Additional Details
### ♦️ Network Interface Card
When choosing a network card for your build, it is often said that spending the extra and going for a genuine Intel-branded NIC can save you a lot of trouble in the process. This is true. However, an Intel NIC such as the X540-T2 may not make total sense depending on your budget and needs. Not to mention navigating a market of counterfeit NICs may not be the experience you're looking for.

This is not to say that other brands or manufacturers of NICs simply make devices that won't work with pfSense or FreeBSD which is the underlying operating system. Quite the contrary, plenty of NICs out there, even the rather cheap Realtek TG-3468 can work. You may just need to be ready for potential curveballs as was the case with the HP332T used in this guide.


### ♦️ SMBus Issue with Intel H110 Chipset
During the initial boot up phase, the computer would go through the standard POST process, followed by the BIOS screen. It is around here where the computer would appear to encounter problems with displaying the BIOS splashscreen when the HP332T network card was installed. Otherwise, the computer would boot up just fine. After much research, this issue seems to have stemmed from an SMBus issue.

The SMBus or System Management Bus is a two-wire interface protocol used for low-speed communication between various components within a computer system especially those on a motherboard. During the boot up phase, computer components communicate with each other to establish identities and intiate as well as complete "handshake" processes in order to boot into Windows successfully.

The Intel H110 Chipset has a specific bug where the boot up process of the motherboard it is part of gets interfered by the boot up process of another device (in this case, the HP332T NIC). This leads to a stall of sorts and the computer is not able to proceed further.


### ♦️ Solution
The solution? We would need to prevent this clash of communication during boot by either making one of the devices recognize the other and apply a compensatory action (which is far too complicated to even attempt as we don't write drivers for any of these devices) or we could simply block them from "seeing" each other. What I mean is, by simply taping up the bins B5 & B6 on the PCIe connector of the HP332T NIC (these are used for communication through SMBus) the computer is now able to go through the booting process with no issues. Interestingly enough, the HP332T doesn't mind not getting a response from the motherboard, but such are the quirks of server-specific hardware.


### ♦️ Resources
**PC not booting with specific PCIe network card**
: https://www.reddit.com/r/techsupport/comments/x6rujd/pc_not_booting_with_specific_pcie_network_card/

**Discussion on overclock.net**
: https://www.overclock.net/threads/perc-5-i-raid-card-tips-and-benchmarks.359025/

**Solution for HP331T**
: https://forums.servethehome.com/index.php?threads/hp-331t-network-adapter-on-asus-p9d-m-motherboard.4254/
