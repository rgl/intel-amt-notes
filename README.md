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
git checkout '8f6cb8ffb7944df5f98c711fabd1f16c6edcbf0b' # 2022-04-29T17:01:58Z
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

### TLS Certificate

Install [create-amt-null-signed-csr](https://github.com/rgl/create-amt-null-signed-csr):

```bash
wget -qO- https://github.com/rgl/create-amt-null-signed-csr/releases/download/v0.0.20220501/create-amt-null-signed-csr_0.0.20220501_linux_amd64.tgz \
  | sudo tar xz -C /usr/local/bin/ create-amt-null-signed-csr
```

Configure the machine with a TLS certificate:

See https://www.youtube.com/watch?v=PNpQV6C0Gb8.

```bash
# make sure there are no pki configurations and tls is disabled.
# NB the next instructions assume there are no certificates and keys installed and tls is disabled.
amtctrl test pki list keys
amtctrl test pki list certs
amtctrl test tls status

# tell the amt device to generate a 2048-bit rsa key pair.
# NB the private key never leaves the device.
# NB key length is limited to 2048-bit. query for the AMT_PublicKeyManagementCapabilities class to see more details.
# NB this returns something like (the second tuple element is the key instance-id):
#     (0, 'Intel(r) AMT Key: Handle: 0')
# TODO make the "amtctrl test pki generate" implementation return the actual RSA public key instead of just returning the reference to the key.
# see Certificate Management at https://software.intel.com/sites/manageability/AMT_Implementation_and_Reference_Guide/WordDocuments/certificatemanagement.htm
# see Enroll a Certificate at https://software.intel.com/sites/manageability/AMT_Implementation_and_Reference_Guide/WordDocuments/enrollacertificate1.htm
# see Transport Layer Security at https://software.intel.com/sites/manageability/AMT_Implementation_and_Reference_Guide/WordDocuments/transportlayersecurity.htm
amtctrl test pki generate 2048

# list the device rsa key pairs and show their corresponding public key.
amtctrl test pki list keys | tail -n +2 >amt-public-key.pem
openssl rsa -noout -text -RSAPublicKey_in -in amt-public-key.pem

# create the null signed csr.
# NB to use openssl 3 instead, see the OpenSSL 3 section in this document.
create-amt-null-signed-csr -pk amt-public-key.pem -cn 192.168.1.89 -ip 192.168.1.89 -dns dm1.test >amt-null-signed-csr.pem
create-amt-null-signed-csr -pk amt-public-key.pem -cn 192.168.1.89 >amt-null-signed-csr.pem
# show the null signed csr content.
# NB verify that the public key is the same as amt-public-key.pem.
openssl req -text -noout -in amt-null-signed-csr.pem
# show the csr raw content.
openssl asn1parse -i -dump -in amt-null-signed-csr.pem

# ask the device to create a proper csr based on the null csr.
amtctrl test pki request amt-null-signed-csr.pem 'Intel(r) AMT Key: Handle: 0' | tail -n +2 >amt-csr.pem
# show the csr content.
# NB verify that the public key is the same as the one in amt-public-key.pem
# NB verify that the verify is successful (Certificate request self-signature verify OK).
openssl req -verify -text -noout -in amt-csr.pem
# show the csr raw content. notice that the public key is the same as amt-public-key.pem.
openssl asn1parse -i -dump -in amt-csr.pem

# create an example test amt ca.
openssl genrsa -out test-amt-ca-key.pem 2048
openssl req -new -sha256 -subj '/CN=Test AMT CA' -key test-amt-ca-key.pem -out test-amt-ca-csr.pem
openssl x509 -req -sha256 \
  -signkey test-amt-ca-key.pem \
  -extensions a \
  -extfile <(echo "[a]
    basicConstraints=critical,CA:TRUE,pathlen:0
    keyUsage=critical,digitalSignature,keyCertSign,cRLSign
    ") \
  -days 365 \
  -in  test-amt-ca-csr.pem \
  -out test-amt-ca-crt.pem
openssl x509 -noout -text -in test-amt-ca-crt.pem

# create the amt device certificate signed by our example ca.
# NB make sure you modify the subjectAltName line.
openssl x509 -req -sha256 \
  -CA test-amt-ca-crt.pem \
  -CAkey test-amt-ca-key.pem \
  -CAcreateserial \
  -extensions a \
  -extfile <(echo "[a]
      subjectAltName=IP:192.168.1.89,DNS:dm1.test
      extendedKeyUsage=critical,serverAuth
      ") \
  -days 365 \
  -in  amt-csr.pem \
  -out amt-crt.pem
# show the crt content.
openssl x509 -noout -text -in amt-crt.pem

# add the certificate to the amt device.
# NB this should return something like:
#     (0, 'Intel(r) AMT Certificate: Handle: 0')
amtctrl test pki add cert amt-crt.pem

# list the installed certificates in the amt device.
# NB this should display something like:
#     {'ElementName': 'Intel(r) AMT Certificate',
#      'InstanceID': 'Intel(r) AMT Certificate: Handle: 0',
#      'Issuer': 'CN=Example AMT CA',
#      'Subject': 'CN=192.168.1.89',
#      'TrustedRootCertficate': 'false'}
#     -----BEGIN CERTIFICATE-----
#     ...
#     -----END CERTIFICATE-----
amtctrl test pki list certs

# enable tls with the given certificate InstanceID.
amtctrl test pki tls 'Intel(r) AMT Certificate: Handle: 0'
amtctrl test tls enable --remote --allow-plaintext
# NB this should return something alike:
#     ('credentials',
#      'TLSProtocolEndpoint Instances Collection',
#      {'ElementInContext': {'Address': 'http://schemas.xmlsoap.org/ws/2004/08/addressing/role/anonymous',
#                            'ReferenceParameters': {'ResourceURI': 'http://intel.com/wbem/wscim/1/amt-schema/1/AMT_PublicKeyCertificate',
#                                                    'SelectorSet': {'Selector': 'Intel(r) AMT Certificate: Handle: 0'}}},
#       'ElementProvidingContext': {'Address': 'http://schemas.xmlsoap.org/ws/2004/08/addressing/role/anonymous',
#                                   'ReferenceParameters': {'ResourceURI': 'http://intel.com/wbem/wscim/1/amt-schema/1/AMT_TLSProtocolEndpointCollection',
#                                                           'SelectorSet': {'Selector': 'TLSProtocolEndpoint Instances Collection'}}}})
#     ('config',
#      'remote',
#      {'AcceptNonSecureConnections': 'true',
#       'ElementName': 'Intel(r) AMT 802.3 TLS Settings',
#       'Enabled': 'true',
#       'InstanceID': 'Intel(r) AMT 802.3 TLS Settings',
#       'MutualAuthentication': 'false'})
#     ('config',
#      'local',
#      {'AcceptNonSecureConnections': 'true',
#       'ElementName': 'Intel(r) AMT LMS TLS Settings',
#       'Enabled': 'false',
#       'InstanceID': 'Intel(r) AMT LMS TLS Settings',
#       'MutualAuthentication': 'false'})
amtctrl test tls status

# test the tls connection.
# NB this should return something like:
#     CONNECTED(00000003)
#     depth=1 CN = Test AMT CA
#     verify return:1
#     depth=0 CN = 192.168.1.89
#     verify return:1
#     ---
#     Certificate chain
#      0 s:CN = 192.168.1.89
#        i:CN = Test AMT CA
#        a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
#        v:NotBefore: Apr 30 21:24:30 2022 GMT; NotAfter: Apr 30 21:24:30 2023 GMT
#     ---
#     Server certificate
#     -----BEGIN CERTIFICATE-----
#     ...
#     -----END CERTIFICATE-----
#     subject=CN = 192.168.1.89
#     issuer=CN = Test AMT CA
#     ---
#     No client certificate CA names sent
#     ---
#     SSL handshake has read 936 bytes and written 636 bytes
#     Verification: OK
#     ---
#     New, TLSv1.2, Cipher is AES128-GCM-SHA256
#     Server public key is 2048 bit
#     Secure Renegotiation IS NOT supported
#     Compression: NONE
#     Expansion: NONE
#     No ALPN negotiated
#     SSL-Session:
#         Protocol  : TLSv1.2
#         Cipher    : AES128-GCM-SHA256
#         Session-ID: 2CB78FFC
#         Session-ID-ctx:
#         Master-Key: 8D70886E363D5BABB4F3DD78B86D2679A21EFDC6ED3EE071D34C80620773EABB5164F16C3CE4ABE5198A045AE7D88CEA
#         PSK identity: None
#         PSK identity hint: None
#         SRP username: None
#         Start Time: 1651354523
#         Timeout   : 7200 (sec)
#         Verify return code: 0 (ok)
#         Extended master secret: no
#     ---
#     DONE
# NB the returned certificate is exactly the same as the one in amt-crt.pem.
# NB my amt device ignores the sni extension (the -servername argument value).
# NB openssl 3: you have to use -legacy_server_connect because by default
#    openssl 3 refuses to connect to the AMT service with:
#       4077E1D6547F0000:error:0A000152:SSL routines:final_renegotiate:unsafe legacy renegotiation disabled:ssl/statem/extensions.c:879:
#    see https://www.openssl.org/docs/man3.0/man3/SSL_get_secure_renegotiation_support.html#SECURE-RENEGOTIATION
(printf 'GET /404 HTTP/1.0\r\n\r\n'; sleep .1) \
  | openssl s_client \
      -CAfile test-amt-ca-crt.pem \
      -connect 192.168.1.89:16993 \
      -servername dm1.test

# add the test-tls device to our hostdb.
amthostdb set -U admin -S https --tls-ca test-amt-ca-crt.pem test-tls 192.168.1.89 HeyH0Password!
amthostdb list
amtctrl test-tls version
amtctrl test-tls power status

# destroy everything.
amtctrl test tls disable --remote --local
amtctrl test pki tls ''
amtctrl test pki rm key 'Intel(r) AMT Key: Handle: 0'
amtctrl test pki rm cert 'Intel(r) AMT Certificate: Handle: 0'
```

#### Build OpenSSL 3

Build OpenSSL 3:

```bash
# create a null signed csr (in pkcs #10 format) using openssl 3 (built from source).
# NB the csr subject must be for the device fqdn.
# NB a null signed csr is a normal csr where its signature will not be
#    verified by the device (because we do not have the device private key).
# NB the csr must include the device public key. this difers from a normal csr
#    where the subject creates a csr for himself, that is, for his own public
#    key and signed with his own private key.
# see https://tools.ietf.org/html/rfc2986
# see https://www.openssl.org/docs/man1.1.1/man1/req.html
# see https://www.openssl.org/docs/man1.1.1/man1/x509.html
# build openssl 3.
wget https://www.openssl.org/source/openssl-3.0.2.tar.gz
tar xf openssl-3.0.2.tar.gz
pushd openssl-3.0.2
./Configure --prefix="$PWD/.local" no-shared
perl configdata.pm --dump
time make install # this takes about 10m in my machine.
alias openssl="$PWD/.local/bin/openssl"
openssl version # OpenSSL 3.0.2 15 Mar 2022 (Library: OpenSSL 3.0.2 15 Mar 2022)
popd
```

Create the null signed csr:

```bash
# convert the amt generated rsa public key (pkcs #1) into a generic public key (pkcs #8).
# NB replace the stdin data with the key that was returned by amtctrl test pki list keys command.
openssl rsa -RSAPublicKey_in -pubin -in amt-public-key.pem -out amt-public-key-pkcs8.pem
# generate a temporary key pair to be able to use the openssl tool to create the null signed csr.
openssl genrsa -out tmp-rsa-key.pem 2048
# show the generated temporary key pair.
openssl rsa -noout -text -in tmp-rsa-key.pem
# show the generated temporary public key.
openssl rsa -pubout -in tmp-rsa-key.pem 2>/dev/null | openssl rsa -noout -text -pubin
# show the amt device public key.
openssl rsa -noout -text -pubin <amt-public-key-pkcs8.pem
# generate the amt null signed csr.
openssl x509 -x509toreq -new -sha256 -subj '/CN=192.168.1.89' -signkey tmp-rsa-key.pem -force_pubkey amt-public-key-pkcs8.pem -out amt-null-signed-csr.pem
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

* [Open AMT Cloud Toolkit](https://github.com/open-amt-cloud-toolkit).
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

* Go:
  * [open-amt-cloud-toolkit/rpc-go](https://github.com/open-amt-cloud-toolkit/rpc-go).
  * [open-amt-cloud-toolkit/go-wsman-messages](https://github.com/open-amt-cloud-toolkit/go-wsman-messages).
* Node.js:
  * [open-amt-cloud-toolkit/wsman-messages](https://github.com/open-amt-cloud-toolkit/wsman-messages).
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
