Overview
=============

Many of the implementation details presented in this book were
informed by Aether, an open source edge cloud that includes a 5G
connectivity service.  Source code for all the individual components
(e.g., AMP, SD-Core, SD-RAN, SD-Fabric) can be downloaded, and
deployment artifacts built from that source code (e.g., Docker Images,
Helm Charts, Fleet Bundles, Terraform Templates) can be used to bring
up a running instance of Aether on local hardware. (See the *Source
Directory* section of this appendix for information about where to
find the relevant code repositories.)

A multi-site deployment of Aether has been running since 2020 in
support of the *Pronto Project*, but that deployment depends on an ops
team with significant insider knowledge about Aether's engineering
details. It is difficult for individuals to reproduce that know-how
and bring up their own Aether clusters.  Aether is also available as
two self-contained software packages that were originally designed to
support developers working individual components.  These packages are
straightforward to install and run, even in a VM on your laptop, so
they also provide a way to get started:

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
Hands-on Programming appendix of our companion SDN book.
  
.. _reading_pronto:
.. admonition:: Further Reading

   `Pronto Project: Building Secure Networks Through Verifiable
   Closed-Loop Control <https://prontoproject.org/>`__.

   `Hands-on Programming (Appendix). Software-Defined Networks: A
   Systems Approach
   <https://sdn.systemsapproach.org/exercises.html>`__. November 2021.

As a tool targeted at developers, AiaB and RiaB support a streamlined
modify-build-test loop, but a significant gap remains between this
these self-contained versions of Aether and an operational 5G-enabled
edge cloud deployed into a particular target environment. `Aether
OnRamp <https://github.com/SystemsApproach/aether-onramp>`__ is a
re-packaging of Aether to address that gap. It provides an incremental
path for users to:

* Learn about all the moving parts in Aether.
* Customize Aether for different target environments.
* Deploy and operate Aether with live traffic.

Aether OnRamp starts with AiaB, refactored to help users step through
a sequence of increasingly complex *blueprints*. The goal is to
prescribe a (mostly) linear sequence of steps a new user can follow to
bring up an operational system that runs 24/7 and supports live 5G
workloads.  Aether OnRamp is still a work in progress, but anyone
interested in participating in that effort is encouraged to join the
discussion on Slack in the `ONF Community Workspace
<https://onf-community.slack.com/>`__. A roadmap for the work that
needs to be done can be found in the `Aether OnRamp Wiki
<https://github.com/SystemsApproach/aether-onramp/wiki>`__.

