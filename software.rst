About The Software
===================

Many of the implementation details presented in this book were
informed by Aether, an open source edge cloud that includes a 5G
connectivity service.  Source code for all the individual components
(e.g., AMP, SD-Core, SD-RAN, SD-Fabric) can be downloaded and
inspected, and deployment artifacts built from that source code (e.g.,
Docker Images, Helm Charts, Fleet Bundles, Terraform Templates) can be
used to bring up a running instance of Aether on local hardware. (See
the *Source Directory* at the end of this appendix for information
about where to find the relevant code repositories.)

A multi-site deployment of Aether has been running since 2020 in
support of the *Pronto Project*, but that deployment depends on a ops
team with significant insider knowledge about Aether's engineering
details. It is difficult for individuals to reproduce that know-how
and bring up their own Aether clusters.  As an alternative, Aether is
also available as two self-contained software packages that are
reasonably straightforward to install and run, even in a VM on your
laptop. These packages were originally designed to support developers
working on Aether components, but they also serve as a way to learn
more about Aether. The two packages are:

* `Aether-in-a-Box (Aiab)
  <https://docs.aetherproject.org/master/developer/aiab.html>`__:
  Includes SD-Core and the online aspects of AMP (Service
  Orchestrator and the Monitoring Subsystem). AiaB can be configured
  to work with either an emulated RAN or physical small cell radios
  (both 4G and 5G).

* `SDRAN-in-a-Box (RiaB)
  <https://docs.sd-ran.org/master/sdran-in-a-box/README.html>`__:
  Includes the ONOS-based nRT-RIC, the O-RAN defined E2SM-KPI and
  E2SM-RC Service Models, and example xApps. RiaB can be configured to
  work with either an emulated RAN or with OAI's open source RAN stack
  running on USRP devices.

Note that these two packages do not include SD-Fabric, which depends
on programmable switching hardware. Readers interested in learning
more about that capability (including a P4-based UPF) should see the
`Hands-on Programming
<https://sdn.systemsapproach.org/exercises.html>`__ appendix of our
companion SDN book.
  
.. _reading_pronto:
.. admonition:: Further Reading

    `Pronto Project: Building Secure Networks Through Verifiable
    Closed-Loop Control <https://prontoproject.org/>`__.

    N. Foster, *et al*. `Using Deep Programmability to put Network Owners
    in Control <https://dl.acm.org/doi/10.1145/3431832.3431842>`__.
    ACM SIGCOMM Computer Communications Review, October 2020.

Both AiaB and RiaB make it possible to gain hands-on experience with
the components described in this book, but a significant gap remains
between these limited versions of Aether and an operational
deployment of a 5G-enabled edge cloud.  To address that gap, there is
an ongoing effort to re-package Aether in a way that provides an
incremental path for users to:

* First learn about and play with Aether;
* Then develop for and contribute to Aether; 
* And finally deploy and operate Aether.

The new packaging, called `Aether OnRamp
<https://github.com/llpeterson/aether-onramp>`__, is derived from
AiaB, refactored to help users step through a sequence of increasingly
complex configurations. The goal of OnRamp is to prescribe a (mostly)
linear sequence of steps a new user can follow to bring up an
operational system that runs 24/7 and supports live 5G workloads.
Aether OnRamp is still a work in progress, but anyone interested in
participating in that effort is encouraged to join the discussion on
Slack in the `ONF Community Workspace
<https://onf-community.slack.com/>`__. A roadmap for the work that
needs to be done can be found in the `Aether OnRamp Wiki
<https://github.com/llpeterson/aether-onramp/wiki>`__.

In the meantime, Stage 1 of Aether OnRamp exists today, and provides a
good way to get started. The following assumes a low-end server that
meets the following requirements:

* Haswell CPU (or newer), with at least 4 CPUs and 12GB RAM.
* Clean install of Ubuntu 18.04, 20.04, or 22.04, with 4.15 (or later) kernel.

While this appendix focuses on deploying Aether OnRamp on a physical
server, Stage 1 can also run in a VM. Options include an AWS VM
(Ubuntu 20.04 image on `t2.xlarge` instance); a VirtualBox VM running
`bento/ubuntu-20.04` `Vagrant <https://www.vagrantup.com>`_ box on
Intel Mac; or a VM created using `Multipass <https://multipass.run>`_
on Linux, Mac, or Windows.

Prep Environment
---------------------

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
-------------------------------

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
-------------------

The first step is to bring up an RKE2.0 Kubernetes cluster on your
target server. Do this by typing:

.. literalinclude:: code/infra.sh 

`kubectl` will show the `kube-system` and `calico-system` namespaces
running.

Connect Kubernetes to the Network
--------------------------------------

Since Aether ultimately provides a connectivity service, how the
cluster you just installed connects to the network is an important
detail. As a first pass, Aether OnRamp borrows a configuration from
AiaB; support for other options, including SR-IOV, will be added over
time.  Type:

.. literalinclude:: code/net.sh 

This target configures Linux (via `systemctl`), but also starts a
Quagga router running inside the cluster.

Bring Up Aether Management Platform
--------------------------------------

The runtime management of Aether is implemented by two Kubernetes
applications: *Runtime Control (ROC)* and a *Monitoring
Service*. (Note that what the implementation calls ROC, Chapter 6
refers to generically as *Service Orchestration*.) The two management
servics can be deployed on the same cluster with the following two
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
-------------------------

We are now ready to bring up the 5G version of the SD-Core:

.. literalinclude:: code/core.sh 

`kubectl` will show the `omec` namespace running. (For historical
reasons, the Core is called `omec` instead of `sd-core`).

Run Emulated RAN Test
---------------------------------

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
-----------------

Working in reverse order, the following Make targets tear down the
three applications you just installed, restoring the base Kubernetes
cluster (plus Quagga router):

.. literalinclude:: code/clean-apps.sh

If you want to also tear down Kubernetes for a fresh install, type:

.. literalinclude:: code/clean-infra.sh

Source Directory
--------------------------

Source code for Aether (and its subsystems) is distributed across
multiple repositories:

* Gerrit repository for the CORD Project
  (https://gerrit.opencord.org): AMP-related components, including
  source for the jobs that implement the CI/CD pipeline.

* GitHub repository for the OMEC Project 
  (https://github.com/omec-project): Microservices for SD-Core. 

* GitHub repository for the ONOS Project
  (https://github.com/onosproject): Microservices for most of
  SD-Fabric and SD-RAN, along with YANG models used to generate the
  ROC API.

* GitHub repository for the Stratum Project
  (https://github.com/stratum): On-switch components of SD-Fabric.
  
For Gerrit, you can either browse Gerrit (select the `master` branch)
or clone the corresponding *<repo-name>* by typing:

.. literalinclude:: code/gerrit.sh

Deployment artifacts are pulled from the following repositories:

Helm Charts

 | https://charts.aetherproject.org
 | https://charts.onosproject.org
 | https://charts.opencord.org
 | https://charts.atomix.io
 | https://sdrancharts.onosproject.org                 
 | https://charts.rancher.io/

Docker Images

 | https://hub.docker.com/u/aetherproject

The Aether CI/CD pipeline, which keeps the above artifact repos in
sync with the source repos, can be found here:

 | ROC: https://gerrit.opencord.org/plugins/gitiles/roc-helm-charts
 | SD-RAN: https://github.com/onosproject/sdran-helm-charts
 | SD-Core: https://gerrit.opencord.org/plugins/gitiles/sdcore-helm-charts
 | SD-Fabric (Servers): https://github.com/onosproject/onos-helm-charts  
 | SD-Fabric (Switches): https://github.com/stratum/stratum-helm-charts

The Jenkins Jobs that implement the CI/CD pipeline are checked into:

 | https://gerrit.opencord.org/plugins/gitiles/aether-ci-management 

These jobs, in turn, depend on QA tests that are checked into:

 | https://gerrit.opencord.org/plugins/gitiles/aether-system-tests 

For more information about Aether's CI/CD pipeline, including its QA
and version control strategies, we recommend the Lifecycle Management
chapter of our companion Edge Cloud Operations book.

.. _reading_cicd:
.. admonition:: Further Reading

    L. Peterson, A. Bavier, S. Baker, Z. Williams, and B. Davie. `Edge
    Cloud Operations: A Systems Approach
    <https://ops.systemsapproach.org/lifecycle.html>`__. June 2022.
