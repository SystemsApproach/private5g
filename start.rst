
Stage 1: Getting Started
---------------------------

The first stage of Aether OnRamp provides a good way to get
started. The following assumes a low-end server that meets the
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

.. literalinclude:: code/firewall.sh 

The first command should report inactive, and the second two commands
should return blank configurations.

Because the install process fetches artifacts from the Internet, if you
are behind a proxy you will need to set the standard Linux environment
variables: `http_proxy`, `https_proxy`, `no_proxy`, `HTTP_PROXY`,
`HTTPS_PROXY` and `NO_PROXY` with the appropriate values. You also
need to export `PROXY_ENABLED=true` by typing the following:

.. literalinclude:: code/proxy.sh 

This variable can also be set in your `.bashrc` file to make it
permanent.

Download Aether OnRamp
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once ready, clone the Aether OnRamp repo on this target deployment
machine:

.. literalinclude:: code/clone.sh 

You will then execute the sequence of Makefile targets described in
the rest of this appendix. After each of these steps, run the
following command to verify that the specified set of Kubernetes
namespaces are operational:

.. literalinclude:: code/kubectl.sh 

If you are not familiar with `kubectl` (the CLI for Kubernetes), we
recommend that you start with `Kubernetes Tutorial
<https://kubernetes.io/docs/tutorials/kubernetes-basics/>`__.

Install Kubernetes
~~~~~~~~~~~~~~~~~~~

The first step is to bring up an RKE2.0 Kubernetes cluster on your
target server. Do this by typing:

.. literalinclude:: code/infra.sh 

`kubectl` will show the `kube-system` and `calico-system` namespaces
running.

Connect Kubernetes to the Network
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since Aether ultimately provides a connectivity service, how the
cluster you just installed connects to the network is an important
detail. As a first pass, Aether OnRamp borrows a configuration from
AiaB; support for other options, including SR-IOV, will be added over
time.  Type:

.. literalinclude:: code/net.sh 

This target configures Linux (via `systemctl`), but also starts a
Quagga router running inside the cluster.

Bring Up Aether Management Platform
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The runtime management of Aether is implemented by two Kubernetes
applications: *Runtime Control (ROC)* and a *Monitoring
Service*. (Note that what the implementation calls ROC, Chapter 6
refers to generically as *Service Orchestration*.) The two management
services can be deployed on the same cluster with the following two
Make targets:

.. literalinclude:: code/amp.sh 

The first command brings up ROC and loads it with a data model for the
Aether API. The second command brings up the Monitoring Service
(Grafana running on top of Prometheus) and loads it with a set of
dashboards.

`kubectl` will show the `aether-roc` and `cattle-monitoring-system`
namespaces now running in support of these two services, respectively,
plus new `atomic-runtime` pods in the `kube-system` name space.
Atomix is the scalable Key/Value Store that underlies ROC.

You can access the dashboards for the two subsystems, respectively, at

.. literalinclude:: code/dashboards

More information about the Control and Monitoring dashboards is given
in their respective sections of the Aether Guide. Note that the
programmatic API underlying the Control Dashboard, which was
introduced in Section 6.4, can be accessed at
`http://<server_ip>:31194/aether-roc-api/`.

.. _reading_dashboards:
.. admonition:: Further Reading

   `Aether Control Dashboard <https://docs.aetherproject.org/master/operations/gui.htmll>`__.

   `Aether Monitoring Dashboard <https://docs.aetherproject.org/master/developer/aiabhw5g.html#enable-monitoring>`__.
 
Bring Up SD-Core
~~~~~~~~~~~~~~~~~~~~~~~~~

We are now ready to bring up the 5G version of the SD-Core:

.. literalinclude:: code/core.sh 

`kubectl` will show the `omec` namespace running. (For historical
reasons, the Core is called `omec` instead of `sd-core`).

Run Emulated RAN Test
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can now test SD-Core with emulated traffic by typing:

.. literalinclude:: code/test.sh 

The monitoring dashboard shows two emulated gNBs come online, with
five emulated UEs connecting to them. (Click on the "5G Dashboard"
once you connect to the main page of the monitoring dashboard to see
their progress.)

This make target can be executed multiple times without restarting the
SD-Core.  (Note that `5g-test` runs an emulator that directs traffic
at Aether. It is not part of Aether, per se.)

Clean Up
~~~~~~~~~~~~~~~~~

Working in reverse order, the following Make targets tear down the
three applications you just installed, restoring the base Kubernetes
cluster (plus Quagga router):

.. literalinclude:: code/clean-apps.sh

If you want to also tear down Kubernetes for a fresh install, type:

.. literalinclude:: code/clean-infra.sh
