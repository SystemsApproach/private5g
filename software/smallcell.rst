Stage 3: External Radio
========================

We are now ready to replace the emulated RAN with a physical small
cell and real UEs. Unlike earlier stages of Aether OnRamp that worked
exlusively with 5G, this stage allows either 4G or 5G small cells (but
not both simultaneously). The following instructions are written for
the 5G scenario, but as a general rule, you can substitute "4G" for
"5G" in every command or file name.  Exceptions to that rule are
explicitly noted.

In addition to the physcial server used in previous stages, we now
assume that server and the external radio are connected to the same L2
network and share an IP subnet. This simplifies communication between
the external radio and the UPF running within Kubernetes on the
server.  This is not a hard requirement for all deployments, but
allows us to configure the network by copying files into
`/etc/systemd/network`. Take note of the network interface on your
server that provides L3 connectivity to the external radio; we refer
to this interface using the ``DATA_IFACE`` variable throughout this
section.

Local Configuration Changes
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Unlike earlier stages that took advantage of canned configurations,
adding a physical base station means you will need to account for
specifics of your local environment. Editing various configuration
files is a necessary step in customizing a deployment, and so Aether
OnRamp establishes a simple convention to help manage the process.

Specifically, the ``configs`` directory includes four files:
``latest``, ``local``, ``release-2.0``, and ``release-2.1``.  These
config files are the ultimate source of information for four different
deployment scenarios—each specifies (a) a set of Helm Charts, and (b)
a set of override values for those charts.  Up to this point we have
been using ``latest``; it is the default value for variable ``CHARTS``
in the ``Makefile``. For the purpose of this stage, we will be using
``local``, which is identical to ``local``, except it points to
alternative override value files for the SD-Core, and alternative
model files for the ROC (as explained below).

.. code-block::

   $ cd configs
   $ diff latest local
   22,25c22,25
   < ROC_4G_MODELS  := $(MAKEDIR)/aether-latest/roc-4g-models.json
   < ROC_5G_MODELS  := $(MAKEDIR)/aether-latest/roc-5g-models.json
   < 4G_CORE_VALUES := $(MAKEDIR)/aether-latest/sd-core-4g-values.yaml
   < 5G_CORE_VALUES := $(MAKEDIR)/aether-latest/sd-core-5g-values.yaml
   ---
   > ROC_4G_MODELS  := $(MAKEDIR)/aether-local/roc-4g-models.json
   > ROC_5G_MODELS  := $(MAKEDIR)/aether-local/roc-5g-models.json
   > 4G_CORE_VALUES := $(MAKEDIR)/aether-local/sd-core-4g-values.yaml
   > 5G_CORE_VALUES := $(MAKEDIR)/aether-local/sd-core-5g-values.yaml

As your deployment deviates more and more from the release—either to
account for differences in your target environment or changes you make
to the software being deployed—you can record these changes in
``configs/local`` (or other variants as circumstances dictate). For
the purpose of this stage, you will need to edit ``MakefileVar.mk`` to
set ``CHARTS ?= local``, and in the subsections that follow, you will
edit the ``yaml`` and ``json`` files in ``aether-local`` to reflect
your particular environment. For now, we recommend familiarizing
yourself with ``aether-local/sd-core-5g-alt-values.yaml`` and
``aether-local/roc-5g-models.json`` (or their 4G counterparts).


Prepare UEs 
~~~~~~~~~~~~

5G-connected devices must have a SIM card, which you are responsible
for creating and inserting.  You will need a SIM card writer (which
are readily available for purchase on Amazon) and a legitimate MCC/MNC
pair. For our purposes, we set MCC=315 (US) and MNC=010 (CBRS), but
you should use whatever values are appropriate for your local
environment. You then assign an IMSI and two secret keys to each SIM
card. Throughout this section, we use the following values:

* IMSI: each one is unique, matching pattern ``315010*********`` (15 digits)
* OPc: ``69d5c2eb2e2e624750541d3bbc692ba5``
* Key: ``000102030405060708090a0b0c0d0e0f``

Insert the SIM cards into whatever devices you plan to connect to
Aether.  Be aware that not all phones support the CBRS frequency bands
that Aether uses. Aether is known to work with recent iPhones (11 and
greater) and Google Pixel phones (4 and greater).  CBRS may also be
supported by recent phones from Samsung, LG Electronics and Motorola
Mobility, but these have not been tested. Note that on each phone you
will need to configure ``internet`` as the *Access Point Name (APN)*.

Finally, modify the the ``subscribers`` block of the
``omec-sub-provision`` section in file
``aether-local/sd-core-5g-values.yaml`` to record the IMSI, OPc, and
Key values configured onto your SIM cards. The block also defines a
sequence number that is intended to thwart replay attacks. (As a
reminder, these values go in ``aether-local/sd-core-4g-values.yaml``
if you are using a 4G small cell.) For example, the following code
block adds IMSIs between 315010999912301 and 315010999912303:

.. code-block::

   subscribers:
   - ueId-start: "315010999912301"
     ueId-end: "315010999912303"
     plmnId: "315010"
     opc: "69d5c2eb2e2e624750541d3bbc692ba5"
     key: "000102030405060708090a0b0c0d0e0f"
     sequenceNumber: 135

Bring Up Aether
~~~~~~~~~~~~~~~~~~~~~

You are now ready to bring Aether on-line, but it is safest to start
with a fresh install of Kubernetes, so first type ``make clean`` if
you still have a cluster running from an earlier stage. Then execute
the following two Make targets, but this time you will disable the RAN
emulator and identify the network interface with the corresponding
variable substitutions:

.. code-block::

   $ make node-prep
   $ ENABLE_RANSIM=false DATA_IFACE=<iface> make net-prep

If you plan to work exclusively with physical small cell radios going
forward, feel free to edit the defaults for these variables in
``MakefileVar.mk``

Once Kubernetes is running and the network properly configured, you
are then ready to bring up the SD-Core as before:

.. code-block::

   $ ENABLE_RANSIM=false DATA_IFACE=<iface> make 5g-core

You can verify the installation by running ``kubectl`` just as you did
in Stage 1. You should see all pods with status ``Running``, keeping
in mind that you will see containers that implement the 4G core
instead of the 5G core running in the ``omec`` namespace if you
configured for that scenario.

Note that we postpone bringing up the AMP until we are confident the
SD-Core is running correctly.


Validating Configuration
~~~~~~~~~~~~~~~~~~~~~~~~

Regardless of which core you run, the UPF pod implements the Core's
User Plane. To verify that the UPF is propertly connected to the
network (which is important because the UPF has to connect to the
radio), you can check to see that the Macvlan networks ``core`` and
``access`` are properly configured on your server. This can be done
using ``ifconfig``, and you should see results similar to the
following:

.. code-block::
   
   $ ifconfig core
   core: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
       inet 192.168.250.1  netmask 255.255.255.0  broadcast 192.168.250.255
       ether 16:9d:c1:0f:19:3a  txqueuelen 1000  (Ethernet)
       RX packets 513797  bytes 48400525 (48.4 MB)
       RX errors 0  dropped 0  overruns 0  frame 0
       TX packets 102996  bytes 26530538 (26.5 MB)
       TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   $ ifconfig access
   access: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
       inet 192.168.252.1  netmask 255.255.255.0  broadcast 192.168.252.255
       ether 7a:9f:38:c0:18:15  txqueuelen 1000  (Ethernet)
       RX packets 558162  bytes 64064410 (64.0 MB)
       RX errors 0  dropped 0  overruns 0  frame 0
       TX packets 99553  bytes 16646682 (16.6 MB)
       TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

Understanding why these two interfaces exist is helpful in
troubleshooting your deployment. They enable the UPF to exchange
packets with the gNB (``access``) and the Internet (``core``). But
these two interfaces exist both **inside** and **outside** the UPF.
The above output from ``ifconfig`` shows the two outside interfaces;
``kubectl`` can be used to see what's running inside the UPF, where
``access`` and ``core`` are the last two interfaces shown below:

.. code-block::
   
   $ kubectl -n omec exec -ti upf-0 bessd -- ip addr
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
   3: eth0@if30: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
       link/ether 8a:e2:64:10:4e:be brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 192.168.84.19/32 scope global eth0
       valid_lft forever preferred_lft forever
       inet6 fe80::88e2:64ff:fe10:4ebe/64 scope link
       valid_lft forever preferred_lft forever
   4: access@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
       link/ether 82:b4:ea:00:50:3e brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 192.168.252.3/24 brd 192.168.252.255 scope global access
       valid_lft forever preferred_lft forever
       inet6 fe80::80b4:eaff:fe00:503e/64 scope link
       valid_lft forever preferred_lft forever
   5: core@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
       link/ether 4e:ac:69:31:a3:88 brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 192.168.250.3/24 brd 192.168.250.255 scope global core
       valid_lft forever preferred_lft forever
       inet6 fe80::4cac:69ff:fe31:a388/64 scope link
       valid_lft forever preferred_lft forever

All four are Macvlan interfaces bridged with ``DATA_IFACE``.  There
are two subnets on this bridge: the two ``access`` interfaces are on
192.168.252.0/24 and the two ``core`` interfaces are on
192.168.250.0/24.  It is helpful to think of two links, called
``access`` and ``core``, connecting the hosting server and the UPF.

The ``access`` interface inside the UPF has an IP address of
``192.168.252.3``; this is the destination IP address of
GTP-encapsulated data plane packets from the gNB.  In order for these
packets to find their way to the UPF, they must arrive on the
``DATA_IFACE`` interface and then be forwarded on the ``access``
interface outside the UPF.  (As described later in this section, it is
possible to configure a static route on the gNB to send the GTP
packets to ``DATA_IFACE``.)  Forwarding the packets to the ``access``
interface is done by the following kernel route, which should be
present if your Aether installation was successful:

.. code-block::

   $ route -n | grep "Iface\|access"
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   192.168.252.0   0.0.0.0         255.255.255.0   U     0      0        0 access

The high-level behavior of the UPF is to forward packets between its
``access`` and ``core`` interfaces, while at the same time
removing/adding GTP encapsulation on the ``access`` side.  Upstream
packets arriving on the ``access`` side from a UE have their GTP
headers removed and the raw IP packets are forwarded to the ``core``
interface.  The routes inside the UPF's `bessd` container will look
something like this:

.. code-block::

   $ kubectl -n omec exec -ti upf-0 -c bessd -- ip route
   default via 169.254.1.1 dev eth0
   default via 192.168.250.1 dev core metric 110
   128.105.144.0/22 via 192.168.252.1 dev access
   128.105.145.141 via 169.254.1.1 dev eth0
   169.254.1.1 dev eth0 scope link
   192.168.250.0/24 dev core proto kernel scope link src 192.168.250.3
   192.168.252.0/24 dev access proto kernel scope link src 192.168.252.3

The default route via 192.168.250.1 is directing upstream packets to
the Internet via the ``core`` interface, with a next hop of the
``core`` interface outside the UPF.  These packets undergo source NAT
in the kernel and are sent to the IP destination in the packet.  The
return (downstream) packets undergo reverse NAT and now have a
destination IP address of the UE.  They are forwarded by the kernel to
the ``core`` interface by these rules on the server:

.. code-block::

   $ route -n | grep "Iface\|core"
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   172.250.0.0     192.168.250.3   255.255.0.0     UG    0      0        0 core
   192.168.250.0   0.0.0.0         255.255.255.0   U     0      0        0 core

The first rule above matches packets to the UEs (on 172.250.0.0/16
subnet).  The next hop for these packets is the ``core`` IP address
inside the UPF.  The second rule says that next hop address is
reachable on the ``core`` interface outside the UPF.  As a result, the
downstream packets arrive in the UPF where they are GTP-encapsulated
with the IP address of the gNB.  Inside the UPF these packets will
match a route like this one (see above; ``128.105.144.0/22`` in this case
is the ``DATA_IFACE`` subnet)::

     128.105.144.0/22 via 192.168.252.1 dev access

These packets are forwarded to the ``access`` interface outside the
UPF and out ``DATA_IFACE`` to the gNB.  Recall that we assume that the
gNB is on the same subnet as ``DATA_IFACE``, so in this case it also
has an IP address in the ``128.105.144.0/22`` range.

Note that If you are not finding ``access`` and ``core`` interfaces on
outside the UPF, the following commands can be used to create these
two interfaces manually:

.. code-block:::

    $ ip link add core link <DATA_IFACE> type macvlan mode bridge 192.168.250.3
    $ ip link add access link <DATA_IFACE> type macvlan mode bridge 192.168.252.3

Runtime Control
~~~~~~~~~~~~~~~

Aether defines an API (and associated GUI) for managing connectivity
at runtime. Even though some connectivity parameters are passed
directly to the SD-Core at startup time using Helm Chart overrides,
(e.g., the IMSI-related edits of
``aether-local/sd-core-5g-values.yaml`` described above), others
correspond to abstractions that ROC layers on top of SD-Core, where
file ``aether-local/roc-5g-models.json`` "bootstraps" the ROC database
with an initial set of data (saving you from a laborious GUI session).

To bring up the ROC, you first need to edit
``aether-local/roc-5g-models.json`` to record the same IMSI
information as before, adding or removing ``sim-card`` entries as
necessary:

.. code-block:::

   "imsi-definition": {
         "mcc": "208",
          "mnc": "93",
          "enterprise": 1,
          "format": "CCCNNEESSSSSSSS"
   },
   ...
   "sim-card": [
          {
              "sim-id": "aiab-sim-1",
              "display-name": "SIM 1",
              "imsi": "208930100007487"
          },
   ...

Then type

.. code-block:::

   $ make 5g-roc
   $ make 5g monitoring

To see these initial configuration values using the GUI, open the
dashboard available at `http://<server-ip>:31194`. If you select
``Configuration > Site`` from the drop-down menu at top right, and
click the ``Edit`` icon assoicated with the ``Aether Site`` you can
see (and potentially change) the following values:

* MCC: 315
* MNC: 010

If you make a change to these values click ``Update`` to save them.

Similarly, if you select ``Sim Cards`` from the drop-down menu at top
right, the ``Edit`` icon associated with each SIM cards allows you to
see (and potentially change) the IMSI values associated with each device.

Finally, the registered IMISs can be aggregated into *Device-Groups*
(a ROC abstraction that makes it easier to associated classes of
devices to different Slices) by selecting ``Device-Groups`` from the
drop-down menu at the top right, and adding a new device group.  When
you are done with these edits, select the ``Basket`` icon at top right
and click the ``Commit`` button.

As currently configured, the Device-Group information is duplicated
between ``aether-loca/sd-core-5g-values.yaml`` and
``aether-local/roc-5g-models.json``. This makes it possible to bring
up the SD-Core without the ROC, for example as we just to verifty the
configuration, but it can lead to problems of keeping the two in sync.
As an exercise, you can delete the subscriber-related blocks in the
former, restart the SD-Core, and see that the latter brings the Aether
up in the correct state. Once running, changes should be made via the
ROC (either the GUI or the API).

.. Altnatively, we should try to remove Device Group info from the
   values file.


gNodeB Setup
~~~~~~~~~~~~~~~~~~~~

Once the SD-Core is up and running, we are ready to bring up the
external gNodeB. The details of how to do this depend on the small
cell radio you are using, but we summarize the main issues you need to
address. For examples commonly used with Aeither, we suggest the
following 4G and 5G small cells from the ONF MarketPlace,
respectively:

.. _reading_sercomm:
.. admonition:: Further Reading

   `SERCOMM – SCE4255W-BCS-A5
   <https://opennetworking.org/products/sercomm-sce4255w-bcs-a5/>`__.

    `SERCOMM – SCE5164-B78 INDOOR SMALL CELL
    <https://opennetworking.org/products/sercomm-sce5164-b78/>`__.

The latter (5G gNB) includes a Users Guide. The former (4G eNB) is
documented in the `Aether Guide
<https://docs.aetherproject.org/master/edge_deployment/enb_installation.html>`__.
We use details from the SERCOMM gNB in the following to make the
discussion concrete.

Start by connecting a laptop directly to the LAN port on the small
cell, pointing your laptop's web browser at the device's management
dashboard (``https://10.10.10.189``), and logging in with the provided
credentials (``login=sc_femto``, ``password=scHt3pp``).  From the
dashboard, configure how the small cell connects to the Internet via
its WAN port, either dynamically using DHCP or staically by setting
the device's IP address (``10.76.28.187``) and default gateway
(``10.76.28.1``). Once on the Internet, it should be possible to reach
the management dashboard without being diectly connected to the LAN
port (``https://10.76.28.187``).

Apart from setting various radio parameters (for which we recommend
starting with the defaults), the remaining configuration tasks involve
(1) setting the PLMN on the small cell (``00101``) to match the
MCC/MNC values (``001`` / ``010`` ) specified in the Core; (2)
connecting the small cell to Aether's Control Plane (AMF); and (3)
connecting the small cell to Aether's User Plane (UPF). The AMF should
already be accessible at well-known port ``38412`` on the physical
server, so setting the AMF address to that server's address
(``10.76.28.113``) should be sufficient (assuming the small cell and
physical server on are on the same subnet).

Connecting the small cell to the UPF running at ``192.168.252.3`` is a
bit more site-dependent, but comes down to establishing a route from
the small cell to the ``192.168.253.0/24`` subnet hosted on the Aether
server (``10.76.28.113``). If the small cell provides a means to
install static routes, then a route to destination
``192.168.252.0/24`` via gateway ``10.76.28.113`` will work (this is
the case for the SERCOMM eNB). If the small cell does not allow static
routes (as is the case for the SERCOMM gNB), then ``10.76.28.113`` can
be installed as the default gateway, but doing so requires that the
server also be configured to forward IP packets on to the Internet.


Connecting Devices
~~~~~~~~~~~~~~~~~~~~~~~~~~

Documenting how to configure different types of devices to work
with Aether is work-in-progress, but here are some basic guidelines.

