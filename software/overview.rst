Overview
----------------

Many of the implementation details presented in this book were
informed by Aether, an open source 5G edge cloud connectivity service.
A multi-site deployment of Aether has been running since 2020 in
support of the *Pronto Project*, but that deployment depends on an ops
team with significant insider knowledge about Aether's engineering
details. It is difficult for others to reproduce that know-how and
bring up their own Aether clusters.

`Aether OnRamp <https://github.com/opennetworkinglab/aether-onramp>`__
is a re-packaging of Aether to address that problem. It provides an
incremental path for users to:

* Learn about and observe all the moving parts in Aether.
* Customize Aether for different target environments.
* Experiment with scalable edge communication.
* Deploy and operate Aether with live traffic.

Aether OnRamp begins with a *Quick Start* deployment that is easy to
bring up in a single VM, but then goes on to prescribe a sequence of
steps users can follow to deploy increasingly complex configurations.
These include both emulated and physical RANs, culminating in an
operational Aether cluster capable of running 24/7 and supporting live
5G workloads. (OnRamp includes support for bringing up a 4G version of
Aether connected to physical eNBs, but we postpone a discussion of
that capability until a later section; everything else in this guide
assumes 5G.)

Note that Aether OnRamp does not include SD-Fabric, which depends
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


.. include:: directory.rst

Aether OnRamp is still a work in progress, but anyone
interested in participating in that effort is encouraged to join the
discussion on Slack in the `ONF Community Workspace
<https://onf-community.slack.com/>`__. A roadmap for the work that
needs to be done can be found in the `Aether OnRamp Wiki
<https://github.com/opennetworkinglab/aether-onramp/wiki>`__.
