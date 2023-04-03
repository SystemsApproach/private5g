Stage 1: Emulated RAN
-----------------------

The first stage of Aether OnRamp provides a good way to get
started. It brings up a one-node Kubernetes cluster, deploys all of
the Aether subsystems on that cluster, and runs an emulated workload
against those subsystems. It assumes a low-end server that meets the
following requirements:

* Haswell CPU (or newer), with at least 4 CPUs and 12GB RAM.
* Clean install of Ubuntu 18.04, 20.04, or 22.04, with 4.15 (or later) kernel.

While this appendix focuses on deploying Aether OnRamp on a physical
server (in anticipation of later stages), Stage 1 can also run in a VM.
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

This variable can also be set in your `.bashrc` file to make it
permanent.

Proxy or no-proxy, Stage 1 involves downloading many Docker images and
Helm Charts. If any of the steps described below fail, it may be due
to network delays, in which case re-executing the step is usually all
it takes to make progress.

Download Aether OnRamp
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once ready, clone the Aether OnRamp repo on this target deployment
machine:

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
<https://kubernetes.io/docs/tutorials/kubernetes-basics/>`__.

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
service ports ranging from ``2000`` to ``36767``. We will see both of
these settings come into play as we move to more complex blueprints.

     
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
applications: *Runtime Control (ROC)* and a *Monitoring
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
`kube-system` namespace.  Atomix is the scalable Key/Value Store that
keeps the ROC data model persistent.

You can access the dashboards for the two subsystems, respectively, at

.. code-block::

   http://<server_ip>:31194 
   http://<server_ip>:30950 
   
More information about the Control and Monitoring dashboards is given
in their respective sections of the Aether Guide. Note that the
programmatic API underlying the Control Dashboard, which was
introduced in Section 6.4, can be accessed at
``http://10.76.28.113:31194/aether-roc-api/`` in our example
deployment.

.. _reading_dashboards:
.. admonition:: Further Reading

   `Aether Control Dashboard <https://docs.aetherproject.org/master/operations/gui.html>`__.

   `Aether Monitoring Dashboard <https://docs.aetherproject.org/master/developer/aiabhw5g.html#enable-monitoring>`__.
 
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

We can now test SD-Core with emulated traffic by typing:

.. code-block::

   $ make 5g-test

As the emulation progresses, the monitoring dashboard will show two
emulated gNBs and five emulated UEs come online, with the performance
graph plotting upstream and downstream transfer rates. All of these
indicators go "silent" once the emulation completes, but you can
execute the `5g-test` target multiple times without restarting the
SD-Core to see additional activity.

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

