.. Repositories
.. ---------------

Aether is a complex system, assembled from multiple components
spanning several Git repositories. These include repos for different
subsystems (e.g., AMP, SD-Core, SD-RAN), but also for different stages
of the development pipeline (e.g., source code, deployment artifacts,
configuration specs).  This rest of this section identifies all the
Aether-related repositories, with the OnRamp repos listed at the end
serving as the starting point for anyone that wants to come
up-to-speed on the rest of the system.

.. admonition:: Troubleshooting Hint

  This guide includes *Troubleshooting Hints* like this one. Our first
  hint is to recommend that the guide be followed sequentially. This
  is because each section establishes a milestone that may prove
  useful when you find yourself trying to troubleshoot a problem in a
  later section. For example, isolating a problem with a physical gNB
  is easier if you know that connectivity to the AMF and UPF works
  correctly, which the *Emulated RAN* section helps to establish.

  Our second hint is to join the ``#aether-onramp`` channel of the
  `ONF Workspace <https://onf-community.slack.com/>`__ on Slack, where
  questions about using OnRamp to bring up Aether are asked and
  answered. The ``Troubleshooting`` bookmark for that channel includes
  summaries of known issues.

Source Repos
~~~~~~~~~~~~~~~~

Source code for Aether and all of its subsystems can be found in
the following repositories:

* Gerrit repository for the CORD Project
  (https://gerrit.opencord.org): Microservices for AMP, plus source
  for the jobs that implement the CI/CD pipeline.

* GitHub repository for the OMEC Project
  (https://github.com/omec-project): Microservices for SD-Core, plus
  the emulator (gNBsim) that subjects SD-Core to RAN workloads.

* GitHub repository for the ONOS Project
  (https://github.com/onosproject): Microservices for SD-Fabric and
  SD-RAN, plus the YANG models used to generate the Aether API.

* GitHub repository for the Stratum Project
  (https://github.com/stratum): On-switch components of SD-Fabric.

For Gerrit, you can either browse Gerrit (select the `master` branch)
or clone the corresponding *<repo-name>* by typing:

.. code-block::

  $ git clone ssh://gerrit.opencord.org:29418/<repo-name>

If port 29418 is blocked by your network administrator, you can try cloning
using https instead of ssh:

.. code-block::

  $ git clone https://gerrit.opencord.org/<repo-name>

Anyone wanting to participate in Aether's ongoing development will
want to learn how to contribute new features to these source repos.

Artifact Repos
~~~~~~~~~~~~~~~~

Aether includes a *Continuous Integration (CI)* pipeline that builds
deployment artifacts (e.g., Helm Charts, Docker Images) from the
source code. These artifacts are stored in the following repositories:

Helm Charts

 | https://charts.aetherproject.org
 | https://charts.onosproject.org
 | https://charts.opencord.org
 | https://charts.atomix.io
 | https://sdrancharts.onosproject.org
 | https://charts.rancher.io/

Docker Images

 | https://registry.aetherproject.org

Note that as of version 1.20.8, Kubernetes uses the `Containerd
<https://containerd.io/>`__ runtime system instead of Docker. This is
transparent to anyone using Aether, which manages containers
indirectly through Kubernetes (e.g., using ``kubectl``), but does
impact anyone that directly depends on the Docker toolchain. Also note
that while Aether documentation often refers its use of "Docker
containers," it is now more accurate to say that Aether uses
`OCI-Compliant containers <https://opencontainers.org/>`__.

The Aether CI pipeline keeps the above artifact repos in sync with the
source repos listed above. Among those source repos are the source
files for all the Helm Charts:

 | ROC: https://gerrit.opencord.org/plugins/gitiles/roc-helm-charts
 | SD-RAN: https://github.com/onosproject/sdran-helm-charts
 | SD-Core: https://gerrit.opencord.org/plugins/gitiles/sdcore-helm-charts
 | SD-Fabric (Servers): https://github.com/onosproject/onos-helm-charts
 | SD-Fabric (Switches): https://github.com/stratum/stratum-helm-charts

The QA tests run against code checked into these source repos can be
found here:

 | https://gerrit.opencord.org/plugins/gitiles/aether-system-tests

For more information about Aether's CI pipeline, including its QA and
version control strategy, we recommend the Lifecycle Management
chapter of our companion Edge Cloud Operations book.

.. _reading_cicd:
.. admonition:: Further Reading

    L. Peterson, A. Bavier, S. Baker, Z. Williams, and B. Davie. `Edge
    Cloud Operations: A Systems Approach
    <https://ops.systemsapproach.org/lifecycle.html>`__. June 2022.

OnRamp Repos
~~~~~~~~~~~~~~~~~~~

The process to deploy the artifacts listed above, sometimes
referred to as GitOps, manages the *Continuous Deployment (CD)* half
of the CI/CD pipeline. OnRamp's approach to GitOps uses a different
mechanism than the one the ONF ops team originally used to manage its
multi-site deployment of Aether.  The latter approach has a large
startup cost, which has proven difficult to replicate. (It also locks
you into deployment toolchain that may or may not be appropriate for
your situation.)

In its place, OnRamp adopts minimal Ansible tooling. This makes it
easier to take ownership of the configuration parameters that define
your specific deployment scenario.  The rest of this guide walks you
through a step-by-step process of deploying and operating Aether on
your own hardware.  For now, we simply point you at the collection of
OnRamp repos:

 | Deploy Aether: https://github.com/opennetworkinglab/aether-onramp
 | Deploy 5G Core: https://github.com/opennetworkinglab/aether-5gc
 | Deploy 4G Core: https://github.com/opennetworkinglab/aether-4gc
 | Deploy Management Plane: https://github.com/opennetworkinglab/aether-amp
 | Deploy 5G RAN Simulator: https://github.com/opennetworkinglab/aether-gnbsim
 | Deploy Kubernetes: https://github.com/opennetworkinglab/aether-k8s

It is the first repo that defines a way to integrate all of the Aether
artifacts into an operational system. That repo, in turn, includes the
other repos as submodules. Note that each of the submodules is
self-contained if you are interested in deploying just that subsystem,
but this guide approaches the deployment challenge from an
integrated/end-to-end perspective.

Because OnRamp uses Ansible as its primary deployment tool, a general
understanding of Ansible is helpful (see the suggested reference).
However, this guide walks you, step-by-step, through the process of
deploying and operating Aether, so previous experience with Ansible is
not a requirement. Note that Ansible has evolved to be both a
"Community Toolset" anyone can use to manage a software deployment,
and an "Automation Platform" offered as a service by RedHat. OnRamp
uses the toolset, but not the platform/service.

.. _reading_ansible:
.. admonition:: Further Reading

   `Overview: How Ansible Works <https://www.ansible.com/overview/how-ansible-works>`__.

