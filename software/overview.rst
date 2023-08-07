Overview
----------------

Many of the implementation details presented in this book were
informed by Aether, an open source 5G edge cloud connectivity service.
Source code for all the individual components (e.g., AMP, SD-Core,
SD-RAN, SD-Fabric) can be downloaded, and deployment artifacts built
from that source code (e.g., Docker Images, Helm Charts, Fleet
Bundles, Terraform Templates) can be used to bring up a running
instance of Aether on local hardware. (See the *Source Directory*
section of this guide for information about where to find the relevant
repositories.)

A multi-site deployment of Aether has been running since 2020 in
support of the *Pronto Project*, but that deployment depends on an ops
team with significant insider knowledge about Aether's engineering
details. It is difficult for others to reproduce that know-how and
bring up their own Aether clusters.  Aether is also available as two
self-contained software packages designed to support developers
working on individual components.  These packages are straightforward
to install and run, even in a VM on your laptop, so they also provide
an easy way to get started:

* `Aether-in-a-Box (AiaB)
  <https://docs.aetherproject.org/master/developer/aiab.html>`__:
  Supports developers working on SD-Core and AMP.

* `SDRAN-in-a-Box (RiaB)
  <https://docs.sd-ran.org/master/sdran-in-a-box/README.html>`__:
  Supports developers working on xApps and the ONOS-based nRT-RIC.

Note that these two packages do not include SD-Fabric, which depends
on programmable switching hardware. Readers interested in learning
more about that capability (including a P4-based UPF) should see the
Hands-on Programming appendix of our companion SDN book.

.. _reading_pronto:
.. admonition:: Further Reading

   `Pronto Project: Building Secure Networks Through Verifiable
   Closed-Loop Control <https://prontoproject.org/>`__.

   `Hands-on Programming (Appendix). Software-Defined Networks: A
   Systems Approach
   <https://sdn.systemsapproach.org/exercises.html>`__. November 2021.

As a tool targeted at developers, AiaB and RiaB support a streamlined
modify-build-test loop, but a significant gap remains between these
self-contained versions of Aether and an operational 5G-enabled edge
cloud deployed into a particular target environment. `Aether OnRamp
<https://github.com/opennetworkinglab/aether-onramp>`__ is a
re-packaging of Aether to address that gap. It provides an incremental
path for users to:

* Learn about and observe all the moving parts in Aether.
* Customize Aether for different target environments.
* Experiment with scalable edge communication.
* Deploy and operate Aether with live traffic.

Aether OnRamp begins with a *Quick Start* deployment similar to AiaB,
but then goes on to prescribe a sequence of steps a user can follow to
deploy increasingly complex configurations. These include both
emulated and physical RANs, culminating in an operational Aether
cluster capable of running 24/7 and supporting live 5G workloads.

Note that OnRamp includes support for bringing up a 4G version of
Aether connected to one or more physical eNBs, but we postpone a
discussion of that capability until a later section. Everything else
in this guide assumes 5G.

Aether OnRamp is still a work in progress, but anyone
interested in participating in that effort is encouraged to join the
discussion on Slack in the `ONF Community Workspace
<https://onf-community.slack.com/>`__. A roadmap for the work that
needs to be done can be found in the `Aether OnRamp Wiki
<https://github.com/opennetworkinglab/aether-onramp/wiki>`__.

