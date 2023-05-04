Stage 1: Emulated RAN
-----------------------

The first stage of Aether OnRamp provides a good way to get
started. It brings up a one-node Kubernetes cluster, deploys SD-Core
and the Aether Management Platform (AMP) on that cluster, and runs an
emulated workload (ICMP packets) against those subsystems. It assumes
a low-end server that meets the following requirements:

* Haswell CPU (or newer), with at least 4 CPUs and 12GB RAM.
* Clean install of Ubuntu 18.04, 20.04, or 22.04, with 4.15 (or later) kernel.

While this appendix focuses on deploying Aether OnRamp on a physical
machine (in anticipation of later stages), Stage 1 can also run in a VM.
Options include an AWS VM (Ubuntu 20.04 image on `t2.xlarge`
instance); a VirtualBox VM running `bento/ubuntu-20.04` `Vagrant
<https://www.vagrantup.com>`_ box on Intel Mac; a VM created using
`Multipass <https://multipass.run>`_ on Linux, Mac, or Windows; or
`VMware Fusion <https://www.vmware.com/products/fusion.html>`__
to run a VM on a Mac.

Prep Environment
~~~~~~~~~~~~~~~~~~~~~

To install Aether OnRamp, you must be able able to run `sudo` without
a password, and there should be no firewall running on the server,
which you can verify as follows:

.. code-block::

   $ sudo ufw status
   $ sudo iptables -L
   $ sudo nft list

The first command should report inactive, and the second two commands
should return blank configurations.

Because the install process fetches artifacts from the Internet, if you
are behind a proxy you will need to set the standard Linux environment
variables: `http_proxy`, `https_proxy`, `no_proxy`, `HTTP_PROXY`,
`HTTPS_PROXY` and `NO_PROXY` with the appropriate values. You also
need to export `PROXY_ENABLED=true` by typing the following:

.. code-block::

   $ export PROXY_ENABLED=true

This variable can also be set in your ``~/.bashrc`` file to make it
permanent.

Proxy or no-proxy, Stage 1 involves downloading many Docker images and
Helm Charts. If any of the steps described below fail, it may be due
to network delays, in which case re-executing the step is usually all
it takes to make progress.

Download Aether OnRamp
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once ready, clone the Aether OnRamp repo on this target deployment
server:

.. code-block::

   $ mkdir ~/systemsapproach
   $ cd ~/systemsapproach
   $ git clone https://github.com/systemsapproach/aether-onramp
   $ cd aether-onramp

You will then execute the sequence of Makefile targets described in
the rest of this appendix. After each of these steps, run the
following command to verify that the specified set of Kubernetes
namespaces are operational:

.. code-block::

   $ kubectl get pods --all-namespaces

If you are not familiar with `kubectl` (the CLI for Kubernetes), we
recommend that you start with `Kubernetes Tutorial
<https://kubernetes.io/docs/tutorials/kubernetes-basics/>`__.  And
although not required, you may also want to install `k9s
<https://k9scli.io/>`__, a terminal-based UI that provides a
convenient alternative to `kubectl` for interacting with Kubernetes.

Helm also has a command-line interface that can be helpful in tracking
progress. For example,

.. code-block::

   $ helm repo list

shows the known set of repos you are pulling charts from, and

.. code-block::

   $ helm list --namespace kube-system

shows the version numbers of the charts currently deployed in the
`kube-system` namespace. Many of the Make targets you will execute in
this section are implemented by a combination of `kubectl` and `helm`
calls, so it is helpful to have a general understanding of how they work.

Install Kubernetes
~~~~~~~~~~~~~~~~~~~

The first step is to bring up an RKE2.0 Kubernetes cluster on your
target server. Do this by typing:

.. code-block::

   $ make node-prep

`kubectl` will show the `kube-system` and `calico-system` namespaces
running.

Bringing up Kubernetes involves generating a ``config.yaml`` file that
specifies several configuration parameters for the cluster. After the
``node-prep`` target completes, you can find this file in
``/etc/rancher/rke2/``, and it should look something like this:

.. code-block::

   $ cat /etc/rancher/rke2/config.yaml
   cni: multus,calico
   cluster-cidr: 192.168.84.0/24
   service-cidr: 192.168.85.0/24
   kubelet-arg:
   - --allowed-unsafe-sysctls=net.*
   - --node-ip=10.76.28.113
   pause-image: k8s.gcr.io/pause:3.3
   kube-proxy-arg:
   - --metrics-bind-address=0.0.0.0:10249
   - --proxy-mode=ipvs
   kube-apiserver-arg:
   - --service-node-port-range=2000-36767

Of particular note, this example cluster runs on a server at IP
address ``10.76.28.113`` and we have instructed Kubernetes to allow
service for ports ranging from ``2000`` to ``36767``. We will see both
of these settings come into play as we move to more complex blueprints.


Configure the Network
~~~~~~~~~~~~~~~~~~~~~

Since Aether ultimately provides a connectivity service, how the
cluster you just installed connects to the network is an important
detail. As a first pass, Aether OnRamp borrows a configuration from
AiaB; support for other options, including SR-IOV, will be added over
time.  Type:

.. code-block::

   $ make net-prep

This target configures Linux (via `systemctl`), but also starts a
Quagga router running inside the cluster. To see how routing is set up
for Aether OnRamp (which you will need to understand in later stages),
you may want to inspect `resources/router.yaml`.

Bring Up Aether Management Platform
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The runtime management of Aether is implemented by two Kubernetes
applications: *Runtime Operational Control (ROC)* and a *Monitoring
Service*. (Note that what the implementation calls ROC, Chapter 6
refers to generically as *Service Orchestration*.) The two management
services can be deployed on the same cluster with the following two
Make targets:

.. code-block::

   $ make 5g-roc
   $ make 5g-monitoring

The first command brings up ROC and loads its database with bootstrap
information (e.g., defining a default Aether site). The second command
brings up the Monitoring Service (Grafana running on top of
Prometheus) and loads it with a set of dashboards.

Once complete, `kubectl` will show the `aether-roc` and
`cattle-monitoring-system` namespaces now running in support of these
two services, respectively, plus new `atomix-runtime` pods in the
`kube-system` namespace.  Atomix is the scalable key-value store that
keeps the ROC data model persistent.

You can access the dashboards for the two subsystems, respectively, at

.. code-block::

   http://<host_ip>:31194
   http://<host_ip>:30950

More information about the Control and Monitoring dashboards is given
in their respective sections of the Aether Guide. The programmatic API
underlying the Control Dashboard, which was introduced in Section 6.4,
can be accessed at ``http://10.76.28.113:31194/aether-roc-api/`` in
our example deployment. Also note that we take a closer look at the
ROC, and the role it plays in managing the rest of Aether, in Stage 4.

.. _reading_dashboards:
.. admonition:: Further Reading

   `Aether Control Dashboard <https://docs.aetherproject.org/master/operations/gui.html>`__.

   `Aether Monitoring Dashboard
   <https://docs.aetherproject.org/master/developer/aiabhw5g.html#enable-monitoring>`__.



Bring Up SD-Core
~~~~~~~~~~~~~~~~~~~~~~~~~

We are now ready to bring up the 5G version of the SD-Core:

.. code-block::

   $ make 5g-core

`kubectl` will show the `omec` namespace running. (For historical
reasons, the Core is called `omec` instead of `sd-core`).

In addition, the monitoring dashboard will show an active (green) UPF,
but no base stations or attached devices at this point.  Note that you
will need to click on the "5G Dashboard" sub-page once you connect to
the main monitoring page.

You can also peruse the Control dashboard by starting with the
dropdown menu in the upper right corner. For example, selecting
`Devices` will show the set of UEs registered with Aether, and
selecting `Device-Groups` will show how those UEs are grouped into
aggregates. In an operational environment, these values would be
entered into the ROC through either the GUI or the underlying API. For
the emulated environment we're limiting ourselves to in Stage 1, these
values are loaded from ``blueprints/latest/roc-5g-models.json`` and match
the settings in ``blueprints/latest/sd-core-5g-values.yaml``.

Run Emulated RAN Test
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can now test SD-Core with emulated traffic (ICMP packets) by typing:

.. code-block::

   $ make 5g-test

As the emulation progresses, the monitoring dashboard will show two
emulated gNBs and five emulated UEs come online, with the performance
graph plotting upstream and downstream transfer rates. All of these
indicators go "silent" once the emulation completes, but you can
execute the ``5g-test`` target multiple times without restarting the
SD-Core to see additional activity.

In addition to the monitoring dashboard, the emulation itself outputs
a detailed trace to the terminal, which concludes with the following
lines when successful:

.. code-block::

   ...
   2023-04-20T20:21:36Z [INFO][GNBSIM][Profile][profile2] ExecuteProfile ended
   2023-04-20T20:21:36Z [INFO][GNBSIM][Summary] Profile Name: profile2 , Profile Type: pdusessest
   2023-04-20T20:21:36Z [INFO][GNBSIM][Summary] UEs Passed: 5 , UEs Failed: 0
   2023-04-20T20:21:36Z [INFO][GNBSIM][Summary] Profile Status: PASS

You can modify the emulation parameters by editing the ``5g-ran-sim``
section of ``blueprints/latest/sd-core-5g-values.yaml``; this block is
used to configure the ``gnbsim-0`` pod in the ``omec`` namespace.
Documentation on how to make such configuration changes can be found
in the `gNBsim GitHub repo
<https://github.com/omec-project/gnbsim>`__.


Run Ksniff and Wireshark
~~~~~~~~~~~~~~~~~~~~~~~~~~~

In addition to the trace output generated by the emulation, a good way
to understand the inner working of Aether is to use `Ksniff
<https://github.com/eldadru/ksniff>`__ (a Kubernetes plugin) to
capture packets and display their headers as they flow into and out of
the microservices that implement Aether. Output from Ksniff can then
be fed into `Wireshark <https://www.wireshark.org/>`__.

To install the Ksniff plugin on the server running Aether, you need to
first install ``krew``, the Kubernetes plugin manager. Instructions on
doing that can be found `online
<https://krew.sigs.k8s.io/docs/user-guide/setup/install/>`__. Once
that's done, you can install Ksniff by typing:

.. code-block::

   $ kubectl krew install sniff

You can then run Ksniff in the context of a specific Kubernetes pod by
specifying their namespace and instance names, and then redirecting
the output to Wireshark. If you don't have a desktop environment on
your Aether server, you can either view the output using a simpler
packet analyzer, such as `tshark
<https://www.wireshark.org/docs/man-pages/tshark.html>`__, or by
redirecting the PCAP output in a file and transfer it a desktop
machine for viewing in Wireshark.

For example, the following captures and displays traffic into and out
of the UPF.  Of course, you'll also need to restart the RAN emulator
to generate workload for this tool to capture.

.. code-block::

   $ kubectl sniff -n omec upf-0 -o - | tshark -r -

As another example, you might want to sniff the ``router`` pod to see
how traffic is passed between UEs and the UPF (on the access side) and
between the UPF and the Internet (on the core side). In this case, it
can be helpful to filter the output, for example, by selecting a
specific interface on the router:

.. code-block::

    $ kubectl sniff -n default router -i access-gw -o - | tshark -r -

In this case, ``access-gw`` is the name of the router's access-side
interface (i.e., the N3 interface as defined by 3GPP).

Packet capture is a great way to learn about the SD-Core and other
components, since you can watch them in action. It can also be a
valuable diagnostic tool, which is a topic we return to in later
stages as we bring up more complex configurations.


Clean Up
~~~~~~~~~~~~~~~~~

Working in reverse order, the following Make targets tear down the
three applications you just installed, restoring the base Kubernetes
cluster (plus Quagga router):

.. code-block::

   $ make core-clean
   $ make monitoring-clean
   $ make roc-clean

If you want to also tear down Kubernetes for a fresh install, type:

.. code-block::

   $ make net-clean
   $ make clean

