# Getting Started with nRF9151 in LTE NTN Mode: Integrating with Amarisoft Callbox for Satellite IoT Connectivity

NTN (Non Terrestrial Network) was introduced in 3GPP Release 17, integrating satellites into cellular networks, allowing global coverage for low power devices. Multiple vendors are already serving this market.

Introducing the nRF9151 SIP, Nordic Semiconductor's SiP with LTE-M/NB-IoT and NTN support, featuring low-power design, integrated GPS, and satellite modem.

Amarisoft Callbox: A versatile SDR-based network emulator for simulating cellular including NTN environments, supporting bands like S-band/L-band for satellite testing.

## Purpose

The purpose of this document is to provide step-by-step instructions to bootstrap a nRF9151 based development kit connected to an Amarisoft callbox, focusing on practical setup rather than deep theory.

Development and testing of NTN based applications can be slow, time-consuming and expensive when utilizing live satellite networks. Utilizing emulated systems can significantly aid the development effort.

## Scope

This guide will cover basic connectivity (S-band NTN mode); it will not include advanced features like multi-constellation handover or production deployment. The nRF9151 DK will connect to the Amarisoft callbox in geostationary mode. LEO (low earth orbit) modes are also possible but are not covered in this guide.

## Target Outcome

Users will be able to establish a cellular link between nRF9151 and Amarisoft callbox, and get bidirectional data transfer working.

## Audience

Engineers familiar with embedded systems, RTOS, LTE/NTN protocols, Linux environments.

# System Requirements

## Hardware Requirements

| Hardware Component            | Version            |
|-------------------------------|--------------------|
| NTN capable nRF9151dk         | v1.2.0 ++          |
| Amarisoft Callbox             | Classic / Mini     |
| 2-way RF combiner 800-2500MHz | Single cell setup  |
| SMA cables                    |                    |
| Development PC                | Win/Linux/Mac      |

The main difference between the Amarisoft callbox variants is the number of cells supported, determined by the physical number of SDR radios built in.

For testing NTN applications it can be useful to get a minimum 3-cell variant, (currently named "Amarisoft Classic") as your application most likely will be connecting to both NTN and TN (terrestrial) networks, perhaps both Cat-M1 and NB-IoT standards, and potentially different combinations of LEO/GEO satellites.

## Safety and Regulatory Notes

RF exposure warnings: Ensure lab environment compliance (FCC/IC limits). Only operate using coaxial connections or enclosed in RF chamber. Running with antennas in open could interfere with local cellular networks.

# Hardware Setup

## Unbox and Connect

- 2 x SMA cable from callbox to RF combiner
- SMA cable + Murata tail from RF combiner to nRF9151DK LTE external antenna connector
- Power and ethernet to Amarisoft callbox
- USB from nrf9151dk to development PC
- Callbox is delivered with test SIM cards. Insert one into slot on nRF9151 DK

## Power Up

- Power on Amarisoft callbox
- Connect the nRF9151 development kit to the PC

# Configuring Amarisoft Callbox

## SSH

It is recommended to work on the Amarisoft callbox via SSH from the development PC.

## Install Software Update

First download the latest software from the Amarisoft customer portal (https://extranet.amarisoft.com), then copy the tarball to the callbox and extract.

Example: 
```
scp ~/Downloads/amarisoft.2025-09-19.tar.gz root@<amarisoft ip addr>:/tmp
ssh root@<amarisoft ip addr>
tar xf /tmp/amarisoft.2025-09-19.tar.gz
```
This creates a 2025-09-19 folder. `cd 2025-09-19` and run `./install.sh`.

For now you can answer yes to all questions. After installation reboot may be requested, which should be performed to load updated firmware into the SDR radio.

### ENodeB Configuration

The configuration file we are interested in is located here: `/root/enb/config/enb.cfg`

This is a symbolic link to one of the files in `/root/enb/config/`. These files have descriptive names, like enb-nbiot-catm1.cfg representing a configuration with one Cat-M1 cell and one nb-iot cell.

The file we want to use is this one: `enb-nbiot-ntn.cfg`

To activate it, make the symbolic link `enb.cfg` point to `enb-nbiot-ntn.cfg`:

    rm -f enb.cfg
    ln -s enb-nbiot-ntn.cfg enb.cfg

To make this work with nRF91 we need to make some small modifications to this file, so make a backup of `enb-nbiot-ntn.cfg` first.

Edit the file and locate the lines below:

    /* Uncomment based on UE interpretation of the ECI reference frame */
    /* eci_reference: "ecef_greenwich" */
    ground_position: {

Now uncomment the line containing "eci_reference" so it becomes like below, but do note the added comma:

    /* Uncomment based on UE interpretation of the ECI reference frame */
    eci_reference: "ecef_greenwich",
    ground_position: {

Next enable "use_state_vectors", find the lines below:

        default_ephemeris: "geo",
        /* Send orbital data in the form of ECEF state vectors, only relevant in GEO */
        /* use_state_vectors: true, */
    #elif NTN_MODE == 1
        ul_sync_validity: 20,

Uncomment as follows:

        default_ephemeris: "geo",
        /* Send orbital data in the form of ECEF state vectors, only relevant in GEO */
        use_state_vectors: true,
    #elif NTN_MODE == 1
        ul_sync_validity: 20,


Also make note of the following section of this config file:

    /* This should reflect the actual UE position in order to simulate properly the service link delay */
    ue_position: {
        latitude: 43.295,
        longitude: 5.373,
        altitude: 20
    },
    ue_doppler_shift: true,


This is the expected location of the nRF9151 DK, the latitude, longitude and altitude values will need to be entered into the nRF9151 DK modem firmware so it will correctly determine some delay parameters.

### RF Driver Config

As coaxial connection is used in our setup, some changes need to be made to `/root/enb/config/rf_driver/config.cfg`:

    tx_gain: 90.0, /* TX gain (in dB) */
    rx_gain: 60.0, /* RX gain (in dB) */


Change the gain values to 60 and 0 dB like this:

    tx_gain: 60.0, /* TX gain (in dB) */
    rx_gain: 0.0, /* RX gain (in dB) */


After changing the configuration, restart the LTE service:

    systemctl restart lte.service


## Hardware connection

Connect the nRF9151DK to the coaxial SDR ports on the callbox using appropriate cables via an RF combiner, as shown in picture below:

![Single cell connection setup](images/scell.jpeg)

Consult the User Guide for your callbox for details on connection setup. Note that the RF combiner is required as the callbox has radio TX and RX on different SMA connectors, and the nRF9151 DK only has one connector for both.

# nRF9151 Development Kit

| Software Required      | Link
|------------------------|-------------------
| nrfutil                | https://www.nordicsemi.com/Products/Development-tools/nRF-Util/Download?lang=en#infotabs
| Modem Firmware NTN     | Awaiting public release
| Modem Shell            | Awaiting public release
| Serial Modem           | [nrf9151dk_mfw-2.0.2_sdk-3.1.0.zip](https://nsscprodmedia.blob.core.windows.net/prod/software-and-other-downloads/dev-kits/nrf9151-dk/application-firmware/nrf9151dk_mfw-2.0.2_sdk-3.1.0.zip) (latest release at time of writing)

Download all the software from the table to the development PC.

## nrfutil

nrfutil is a single binary you can run from anywhere, putting it somewhere in your `$PATH` is assumed in the following steps. 

Run the following commands to download the *device* command we will use to load firmware onto the nRF91DK:

    nrfutil self-upgrade
    nrfutil install device

Run the following command after connecting your nRF9151DK via USB and observe output similar to below:

    ~$ nrfutil device list

    1051286708
    Product         J-Link
    Board version   PCA10171
    Ports           /dev/ttyACM0, vcom: 0
                    /dev/ttyACM1, vcom: 1
    Traits          jlink, devkit, seggerUsb, modem, boardController, usb, serialPorts

    Supported devices found: 1
    $> _

If you see output similar to above your device is operating correctly.

## Modem Firmware NTN

From here on, commands assume only one DK connected. If more than one is connected, specify device with `--serial-number` parameter.

Load the NTN enabled modem firmware onto the nRF9151 DK:

    nrfutil device program --firmware path/to/mfw_nrf91x1_ntn-0.5.0.zip

# Applications

## Modem Shell Application

Load the NTN enabled modem shell application:

    nrfutil device program --firmware nrf9151_mosh_ntn_0.1.hex

On the development PC use your favorite terminal emulator and connect to your nRF9151 DK:

    minicom /dev/ttyACM0

Note that development kits usually have multiple ports, but normally the first one is used for application logs.

Press the reset button and you should see output similar to below:

    *** Booting nRF Connect SDK v3.1.99-4102a1851515 ***
    *** Using Zephyr OS v4.2.99-28a66c459624 ***

    Reset reason: software

    mosh:~$
    MOSH version:       v3.1.99-4102a1851515
    MOSH build id:      v125
    MOSH build variant: development/normal_ext_antenna_modem_uart_trace_v125
    HW version:         nRF9151 LACA APA
    Modem FW version:   mfw_nrf9151-ntn_0.5.0
    Modem FW UUID:      4a69a961-c52c-4a1c-8d78-c517a28eafa3


    Network registration status: searching
    Currently active system mode: LTE-M

    mosh:~$ _

This application has a terminal interface. To enable NTN connectivity we need to type the following commands into the terminal:

    link funmode -0
    at AT%CELLULARPRFL=2,0,4,0 
    at AT%CELLULARPRFL=2,1,1,0 
    link sysmode --ntn 
    at AT%LOCATION=2,\"43.295\",\"5.373\",\"20\",0,0  # These are lat/lon values from config file
    at AT+CPSMS=0
    link funmode -1

If you changed the lat/lon values in the Amarisoft enb config file, adjust your `AT%LOCATION` command accordingly.

Within a minute the device should now establish connection:

    Network registration status: searching
    Currently active system mode: NTN NB-IoT
    LTE cell changed: ID: 27447554, Tracking area: 2
    Modem domain event: CE-level 0
    RRC mode: Connected
    PDN event: PDP context 0 activated
    PDN event: PDP context 0, PDN type IPv4 only allowed
    Network registration status: Connected - home network
    Modem domain event: Search done
    PSM parameter update: TAU: 1800, Active time: -1 seconds
    Modem config for system mode: NTN NB-IoT
    Modem config for LTE preference: No preference, automatically selected by the modem
    Battery voltage:       5128 mV
    Modem temperature:     24 C
    Device ID:             50423451-3737-406f-80f9-0f154bb7b53d
    Operator full name:   "Amarisoft Network"
    Operator short name:  "Amarisoft"
    Operator PLMN:        "00101"
    Current cell id:       27447554 (0x01A2D102)
    Current phy cell id:   37
    Current band:          255
    Current TAC:           2 (0x0002)
    Current rsrp:          88: -53dBm
    Current snr:           45: 21dB
    Mobile network time and date: 25/10/01,13:05:07+08
    PDP context info 1:
        CID:                0
        PDN ID:             0
        PDP context active: yes
        PDP type:           IP
        APN:                default
        IPv4 address:       192.168.2.2
        IPv6 address:       ::
        IPv4 DNS address:   8.8.8.8, 0.0.0.0
        IPv6 DNS address:   ::, ::
    RRC mode: Idle

Now we are connected to GEO emulated satellite. Check if a DNS request works:

    mosh:~$ sock getaddrinfo -H google.com
    getaddrinfo query family=0, type=0, pdn_cid=0, hostname=google.com
    getaddrinfo answer family=1, address=142.250.74.78
    RRC mode: Idle
    mosh:~$ _

Check if we can send UDP data to an IP address/port;

    sock connect -I 0 -a 95.217.154.188 -p 21185 -t dgram
    sock send -i 0 -d somebytesofdata

If the IP used above is a typical UDP echo service, output similar to below should be printed:

    mosh:~$ sock send -i 0 -d somebytesofdata
    Socket data send:
            somebytesofdata
    Received data for socket socket_id=0, buffer_size=15:
            somebytesofdata

# Serial Modem

Unzip the firmware package downloaded from the nRF9151DK product page.

    unzip nrf9151dk_mfw-2.0.2_sdk-3.1.0.zip

Load the serial modem application from the extracted package:

    nrfutil device program --firmware img_app_bl/nrf9151dk_serial_lte_modem_2025-08-14_6c6e5b32.hex

On the development PC use your favorite terminal emulator and connect to your nRF9151 DK:

    minicom /dev/ttyACM0

Upon reset of the nRF9151DK the application should print the string `Ready` to the serial terminal.

Currently the serial modem needs a full CRLF at the end of commands to properly start processing. Check your terminals settings to make sure it works. Typing in the string `AT\r\n` should return the string `OK`.

Note that this application does not echo characters back, like modem_shell did above.

After running the following AT commands, the device should be connected to the callbox:

    AT+CFUN=0
    AT%CELLULARPRFL=2,0,4,0
    AT%CELLULARPRFL=2,1,1,0
    AT%XSYSTEMMODE=0,0,0,0,1
    AT%LOCATION=2,"63.421","10.437","160",0,0
    AT%XBANDLOCK=2,,"256"
    AT+CPSMS=0
    AT+CFUN=1

Now you should be able to run the AT command `AT+CGDCONT?` and get output indicating we are connected aand have IP address from callbox:

    +CGDCONT: 0,"IP","test123","192.168.3.2",0,0
    OK

Now SLM AT commands should work, running `AT#XGETADDRINFO="www.google.com"` shoud return something like:

    OK
    #XGETADDRINFO: "216.58.207.228"

# Summary

We now have a working NTN connection between the nRF9151DK and the callbox.

Using the modem_shell or serial_modem application and a cellular NTN operator SIM card, the same procedure should work connecting to a real satellite.

The callbox has a web GUI page that displays logs and device connection information at its IP address: `http://<callbox ip>/`.

