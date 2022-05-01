# About

Notes about [Intel AMT](https://en.wikipedia.org/wiki/Intel_Active_Management_Technology) (aka iAMT, aka Active Management Technology).

## Setup

Run the AMT Configuration Utility as Administrator, click the `Create Settings to Configure Multiple Systems` button:

![](setup-bin-acu-create-settings.png)

Click the `Tools` button, then select the `Prepare a USB Key for Manual Configuration...` tool:

![](setup-bin-acu-settings.png)

This should create the `Setup.bin` (saved in this repo as [Setup-default.bin](Setup-default.bin)) file in the USB drive.

Start MeshCommander, then open the `Setup.bin` file using the `Setup.bin Manager` tab:

![](setup-bin-meshcommander-settings-default.png)

Using the `Add Variable` link add the following variables:

| Variable | Value |
|----------|-------|
| `Manageability Feature Selection` | `Intel AMT` |
| `SOL/IDER Redirection Configuration` | `SOL+IDER - User/Pass Enabled` |

The end result should be:

![](setup-bin-meshcommander-settings.png)

Save the result to the `Setup.bin` file (saved in this repo as [Setup.bin](Setup.bin)) inside the USB drive.

Enter the system BIOS, enable `Intel AMT` and `USB Provisioning of AMT`:

![](bios-enable-amt.png)

Insert the USB drive into the system, save the BIOS settings and reboot the system. At the next boot, the system should ask you to provision AMT from USB. After provisioning you should see something alike:

![](bios-provision-amt.jpg)

Add the computer to MeshCommander and connect to it. You should see something alike:

![](meshcommander-connected-computer.png)

You should now have a basic AMT working. Go ahead and explore it!

## Python Usage (nomis/intel-amt)

Install [nomis/intel-amt](https://github.com/nomis/intel-amt):

```bash
mkdir tmp && cd tmp
sudo apt-get install -y --no-install-recommends python3-pip python3-venv
rm -rf .venv
python3 -m venv .venv
source .venv/bin/activate
cd .venv
git clone https://github.com/nomis/intel-amt.git intel-amt
cd intel-amt
git checkout b33b6e98e51e8af3ae23f883dc7266ed588ad324 # 2022-04-03T13:42:33Z
wget -qO- https://github.com/nomis/intel-amt/pull/1.patch | patch --strip 1 # see https://github.com/nomis/intel-amt/pull/1
python3 setup.py install
```

Try `amtctrl`:

```bash
# add "test" host to ~/.config/amtctrl/hosts.cfg
# NB credentials are stored in cleartext.
amthostdb set -U admin test 192.168.1.89 HeyH0Password!
amthostdb list
amtctrl test version
amtctrl test uuid
amtctrl test user list
amtctrl test power status
amtctrl test boot status
amtctrl test tls status
amtctrl test pki list certs
amtctrl test pki list keys
```

### Power Management

Power on the machine:

```bash
# this can be one of:
#   POWER_STATES = {
#       'on': 2,
#       'standby': 3,
#       'sleep': 4,
#       'reboot': 5,
#       'hibernate': 7,
#       'off': 8,
#       'hard-reboot': 9,
#       'reset': 10,
#       'nmi': 11,
#       'soft-off': 12,
#       'soft-reset': 14,
#   }
# NB depending on the current power state, the machine can only transition to
#    sub-set of the above. unfortunately, amtctrl does not have a way to list
#    the currently available power states (available in the
#    CIM_AssociatedPowerManagementService.AvailableRequestedPowerStates
#    property) AND when something fails amtctrl will just dump the AMT
#    service SOAP response without modifying its exit code. e.g.:
#       ...
#       <g:RequestPowerStateChange_OUTPUT>
#         <g:ReturnValue>2</g:ReturnValue>
#       </g:RequestPowerStateChange_OUTPUT>
#       ...
#    Example power state transitions:
#       | State   | Transition  | Notes                                                                  |
#       |---------|-------------|------------------------------------------------------------------------|
#       | off     | on          |                                                                        |
#       | on      | reset       |                                                                        |
#       | on      | nmi         |                                                                        |
#       | on      | sleep       |                                                                        |
#       | on      | hibernate   |                                                                        |
#       | on      | soft-reset  | graceful reset. only available when the LMS agent is running in the OS |
#       | on      | soft-off    | graceful off. only available when the LMS agent is running in the OS   |
#       | on      | off         | abrupt shutdown. equivalent to pulling the power cable                 |
# NB WHEN REMOTE DESKTOP OR IDER IS ACTIVE, NOT ALL TRANSITIONS ARE POSSIBLE.
#    For example, when on and remote desktop is active, it cannot transition
#    to the off state. when this happens, the return value is 2 (not ready).
amtctrl test power status     # TODO modify this to return the available power state transitions.
amtctrl test power on
amtctrl test power soft-off   # soft power off (graceful shutdown; LMS must be running in the OS).
amtctrl test power off        # hard power off (abrupt shutdown; equivalent to pulling the power cable).
amtctrl test power soft-reset # soft reset (graceful shutdown and then power on).
amtctrl test power reset      # hard reset (abrupt shutdown and then power on; equivalent to pulling the power cable).
```

### Network Booting

Restart the machine into PXE boot:

```bash
amtctrl test pxeboot
```

## Python Usage (openwsman)

**NB** openwsman is too low-level to be practical.

See https://github.com/Openwsman/openwsman/blob/master/bindings/python

Install the pywsman python package (on Ubuntu 20.04 and 22.04 this is only available for python 2):

```bash
sudo apt-get install -y python-openwsman
```

Try it:

```bash
python2 <<'EOF'
import pywsman as wsman

client = wsman.Client('http://admin:HeyH0Password!@192.168.1.89:16992/wsman')
options = wsman.ClientOptions()
options.set_max_elements(20)
#options.set_dump_request()

# show identity.
body_node = client.identify(options).body()
protocol_version = body_node.find(wsman.XML_NS_WSMAN_ID, 'ProtocolVersion')
product_vendor = body_node.find(wsman.XML_NS_WSMAN_ID, 'ProductVendor')
product_version = body_node.find(wsman.XML_NS_WSMAN_ID, 'ProductVersion')
print('Identity: Protocol %s, Vendor %s, Version %s' % (protocol_version, product_vendor, product_version))

# enumerate CIM_SoftwareIdentity.
# see https://software.intel.com/sites/manageability/AMT_Implementation_and_Reference_Guide/default.htm?turl=HTMLDocuments%2FWS-Management_Class_Reference%2FCIM_SoftwareIdentity.htm
ns = 'http://schemas.dmtf.org/wbem/wscim/1/cim-schema/2/CIM_SoftwareIdentity'
body_node = client.enumerate(options, None, ns).body()
context_node = body_node.find(wsman.XML_NS_ENUMERATION, 'EnumerationContext')
while context_node:
  response_node = client.pull(options, None, ns, str(context_node))
  body_node = response_node.body()
  for node in body_node.find(wsman.XML_NS_ENUMERATION, 'Items'):
    instance_id_node = node.get('InstanceID')
    version_node = node.get('VersionString')
    print('CIM_SoftwareIdentity: %s %s' % (instance_id_node, version_node))
  context_node = body_node.find(wsman.XML_NS_ENUMERATION, 'EnumerationContext')
if context_node:
  client.release(options, ns, str(context_node))
EOF
```

## Tools

* https://www.meshcommander.com/
  Open-source tool for managing AMT enabled systems from Linux.
  The source code is at:
  * https://github.com/Ylianst/MeshCommander
  * https://github.com/Ylianst/MeshCentral
* https://www.meshcommander.com/meshcommander/firmware
  This can replace the UI of AMT with MeshCommander.
* https://www.meshcommander.com/meshcommander/meshcmd
  meshcmd can configure amt from the command line.
* Intel Setup and Configuration Software (Intel SCS)
  * it can put a Setup.bin inside a USB pen to automatically configure hosts in Admin Control Mode (ACM).
    * essentially, when the to-be controlled host boots, AMT detects the Setup.bin file
      inside the USB pen and asks the user to apply its settings.
    * **NB** you can further customize this file using the `Setup.bin Manager` tab of MeshCommander.
    * **NB** for this provision method to work you must enable it in the UEFI firmware.
  * https://downloadcenter.intel.com/download/26505/Intel-Setup-and-Configuration-Software-Intel-SCS-
  * https://downloadmirror.intel.com/26505/eng/Configurator_download_package_12.1.0.87.zip
    * Version: 12.1 (Latest) Date: 4/10/2019
    * **NB** this is the last version that I known that lets you create a `Setup.bin` file using the `Prepare a USB Key for Manual Configuration...` tool window.
    * **NB** I've added a [copy of ACUWizardInstaller-12.1.0.87.msi into this repository](ACUWizardInstaller-12.1.0.87.msi).
    * **NB** you might need to disable the malware/anti-virus protection for
             being able to actually create the pen like the SCS tool wants.
* [OpenWSMAN](https://openwsman.github.io/)

## Libraries

* Node.js:
  * [Ylianst/MeshCentral/amt](https://github.com/Ylianst/MeshCentral/tree/master/amt)
    * See the [meshcmd source code?](https://github.com/Ylianst/MeshCommander/issues/17) issue.
* Python:
  * [sdague/amt](https://github.com/sdague/amt)
  * [nomis/intel-amt](https://github.com/nomis/intel-amt)
    * A fork of `sdague/amt` that has more features.
  * [OpenStack Ironic AMT drivers](https://opendev.org/x/ironic-staging-drivers/src/branch/master/ironic_staging_drivers/amt)
    * **NB** These drivers were removed from OpenStack Ocata as described in the [Release Notes](https://docs.huihoo.com/openstack/docs.openstack.org/releasenotes/ironic/ocata.html).

## Documentation

* [Intel vPro Platforms/Intel Active Management Technology (Intel AMT) Lab Setup Guide Video](https://www.intel.com/content/www/us/en/support/articles/000026592/technologies.html)
* [Intel AMT Implementation and Reference Guide](https://software.intel.com/sites/manageability/AMT_Implementation_and_Reference_Guide/default.htm)
* [Active Platform Management Demystified](https://www.meshcommander.com/active-management) is an on-line book about AMT.
