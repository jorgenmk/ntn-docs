# Getting Started with nRF9151 in NTN Mode: Integrating with Amarisoft Callbox for Satellite IoT Connectivity

## Introduction

NTN (Non Terrestrial Network) was introduced in 3GPP Release 17/18, integrating satellites into cellular networks, allowing global coverage for low power devices.

Introducing the nRF9151 SIP, Nordic Semiconductor's SiP with LTE-M/NB-IoT and NTN support, featuring low-power design, integrated GPS, and satellite modem.

Amarisoft Callbox: A versatile SDR-based network emulator for simulating NTN environments, supporting bands like S-band/L-band for satellite testing.

### Purpose

Purpose of this document is to provide step by step instructions to bootstrap a nRF9151 based development kit connected to an Amarisoft callbox, focusing on practical setup rather than deep theory. Development and testing of NTN based applications is slow, time consuming and expensive when utilizing live satellite networks. 

### Scope

This guide will cover basic connectivity (S-band NTN mode); it will not include advanced features like multi-constellation handover or production deployment.

### Target Outcome

User will be able to establish cellular link between nRF9151 and Amarisoft callbox, and get bidirectional data transfer working. UDP /TCP discussion/specification?

### Audience

Engineers familiar with embedded systems, LTE/NTN protocols, Linux environments, RTOS / nRF Connect SDK.

## System Requirements

### Hardware Requirements

| Hardware Component            | Version          |
|-------------------------------|------------------|
| NTN capable nRF9151dk         | v1.2.0 ++        |
| Amarisoft Callbox             | 2025-09-19       |
| 12-way RF combiner 800-2500MHz| Three cells      |
| 2-way RF combiner 800-2500MHz | Single cell      |
| SMA - SMA cables              |                  |
| Murata connector tails        |                  |
| Development PC                | Win/Linux/Mac    |

The main difference between the Amarisoft callbox variants is the number of cells supported, determined by the physical number of SDR radios built in.

For testing NTN applications it can be useful to get at least a 3 cell variant, (currently named "Amarisoft Classic") as your application most likely will be connecting to both NTN and terrestrial networks, and sometimes both catm1 and nb-iot terrestial variants.

### Software Requirements

- nrF Connect SDK
- nrfutil
- serial terminal
- python for automation

### Safety and Regulatory Notes

RF exposure warnings: Ensure lab environment compliance (FCC/IC limits). Only operate using coaxial connections or enclosed in RF chamber. Running with antennas in open could interfere with local cellular networks.

## Hardware Setup

### Unbox and Connect
- 12 x SMA cable from callbox to RF combiner
- SMA cable + Murata tail from RF combiner to nRF9151DK LTE external antenna connector
- Power and ethernet to Amarisoft callbox
- USB to nrf9151dk
- Callbox comes with test SIM cards. Insert one into slot on nRF9151 DK

### Power Up
- Power on Amarisoft with monitor and keyboard attached
- Log in and get the devices IP address eg from `ip a`
- Now you can ssh to the callbox and disconnect monitor/keyboard if desired
- Connect the DK to the development PC, connect to serial and check output

## Configuring Amarisoft Callbox

### Install Software Update

First download the latest software from the Amarisoft customer portal (https://extranet.amarisoft.com), then copy the tarball to the callbox and extract.

Example: 
```
scp ~/Downloads/amarisoft.2025-09-19.tar.gz <amarisoft ip addr>:/tmp
ssh root@<amarisoft ip addr>
tar xf /tmp/amarisoft.2025-09-19.tar.gz
```
This creates a 2025-09-19 folder. `cd 2025-09-19` and run `./install.sh`.

For now answer yes to everything. After installation it can request reboot, which should be performed to load updated firmware into the SDR radio.

### ENodeB Configuration

The configuration we are interested in is located here: `/root/enb/config/enb.cfg`

It is a symbolic link to one of the files in `/root/enb/config/`. These files have descriptive names, like enb-nbiot-catm1.cfg representing a configuration with one catm1 cell and one nb-iot cell.

The file we are intersted in using for our basic configuration is this one: `enb-nbiot-ntn.cfg`

To activate it, make the symbolic link `enb.cfg` point to `enb-nbiot-ntn.cfg`:

```
rm -f enb.cfg
ln -s enb-nbiot-ntn.cfg enb.cfg
```

To make this work with nRF91 we need to make some small modifications to this file.

Edit the file and locate the lines below:
```
    /* Uncomment based on UE interpretation of the ECI reference frame */
    /* eci_reference: "ecef_greenwich" */
    ground_position: {
```
Now uncomment the line containing "eci_reference" so it becomes like below, but do note the added comma:
```      
    /* Uncomment based on UE interpretation of the ECI reference frame */
    eci_reference: "ecef_greenwich",
    ground_position: {
```

Also make note of the following section of the config file:
```
    /* This should reflect the actual UE position in order to simulate properly the service link delay */
    ue_position: {
        latitude: 43.295,
        longitude: 5.373,
        altitude: 20
    },
    ue_doppler_shift: true,
```

This is the expected location of the nrf9151 dk, the latitude, longitude and altitude values will need to be entered into the nRF9151dk.

### RF Driver Config
As coaxial connection is used in our setup, some changes needs to be made to `/root/enb/config/rf_driver/config.cfg`:
```
tx_gain: 90.0, /* TX gain (in dB) */
rx_gain: 60.0, /* RX gain (in dB) */
```

Change the gain values to 60 and 0 dB like this:
```
tx_gain: 60.0, /* TX gain (in dB) */
rx_gain: 0.0, /* RX gain (in dB) */
```

After changing the configuration, restart the LTE service:
```
systemctl restart lte.service
```

## Configure nRF9151 Development Kit

| Software Requirements  | Link
|------------------------|-------------------
| nrfutil                | [Nrfutil Download page](https://www.nordicsemi.com/Products/Development-tools/nRF-Util/Download?lang=en#infotabs)
| Modem Firmware NTN     | https://linky
| Modem Shell Application| https://linky

Download all the software from the table.

### `nrfutil`
Nrfutil is a single binary you can run from anywhere, putting it somewhere in `$PATH` is assumed in next steps.

Run the following commands to prepare `nrfutil`:

```
nrfutil self-upgrade
nrfutil install device
```

Run the following command after connecting your DK via USB and observe output similar to below:
```
$> nrfutil device list

1051286708
Product         J-Link
Board version   PCA10171
Ports           /dev/ttyACM0, vcom: 0
                /dev/ttyACM1, vcom: 1
Traits          jlink, devkit, seggerUsb, modem, boardController, usb, serialPorts

Supported devices found: 1
$> _
```
### Modem Firmware NTN
The following commands assune only one DK connected.

Load the NTN enabled modem firmware onto the nrf9151:
```
nrfutil device program --firmware path/to/mfw_nrf91x1_ntn-0.5.0.zip
````

### Modem Shell Application
Load the NTN enabled modem shell application:
```
nrfutil device program --firmware nrf9151_mosh_ntn_0.1.hex
```



# Missing:
- Public link to ntn mfw
- Public link to mosh ntn build
