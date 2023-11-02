Many of the implementation details presented in this book were
informed by Aether, an open source Private 5G platform. This appendix
serves as a guide for gaining hands-on experience with that software.
Because Aether is an active open source project, we recommend the
`Aether Guide <https:/docs.aetherproject.org>`__ for tracking the
latest updates to the software (and its documentation). Anyone
interested in participating in the project is also encouraged to join
the discussion in the `ONF Community Workspace
<https://onf-community.slack.com/>`__ on Slack.


A.1  Overview
----------------

A multi-site deployment of Aether has been running since 2020 in
support of the *Pronto Project*, but that deployment depends on an
experienced ops team with significant insider knowledge about Aether's
engineering details. It is difficult for others to reproduce that
know-how and bring up their own Aether clusters.  `Aether OnRamp
<https://github.com/opennetworkinglab/aether-onramp>`__ is a packaging
of Aether to address that problem. It provides tooling to deploy
Aether on your own hardware, giving you an incremental path to:

* Learn about and observe all the moving parts in Aether.
* Customize Aether for different target environments.
* Experiment with scalable edge communication.
* Deploy and operate Aether with live traffic.

Aether OnRamp begins with a *Quick Start* deployment that is easy to
bring up in a single VM, but then goes on to prescribe a sequence of
steps users can follow to deploy increasingly complex configurations.
OnRamp refers to each such configuration as a *blueprint*, and the set
supports both emulated and physical RANs, along with the runtime
machinery needed to operate an Aether cluster supporting live 5G
workloads.  (OnRamp also defines a 4G blueprint that can be used to
connect one or more physical eNBs, but we postpone a discussion of
that capability until a later section. Everything else in this guide
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

