Source Directory
--------------------------

Source code for Aether and all of its subsystems is distributed across
multiple repositories:

* Gerrit repository for the CORD Project
  (https://gerrit.opencord.org): Microservices for AMP, plus source
  for the jobs that implement the CI/CD pipeline.

* GitHub repository for the OMEC Project
  (https://github.com/omec-project): Microservices for SD-Core, plus
  the emulator that subjects SD-Core to RAN workloads.

* GitHub repository for the ONOS Project
  (https://github.com/onosproject): Microservices for SD-Fabric and
  SD-RAN, plus the YANG models used to generate the Aether API.

* GitHub repository for the Stratum Project
  (https://github.com/stratum): On-switch components of SD-Fabric.
  
For Gerrit, you can either browse Gerrit (select the `master` branch)
or clone the corresponding *<repo-name>* by typing:

.. code-block::

   $ git clone ssh://gerrit.opencord.org:29418/<repo-name>

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

Finally, Aether OnRamp—the focus of this appendix—defines one possible
way to integrate all of the above artifacts into an end-to-end system
that can be deployed and operated with live traffic. OnRamp is
available on GitHub:

 | https://github.com/SystemsApproach/aether-onramp

As described in the following sections, OnRamp prescribes a
step-by-step process for growing an Aether deployment from a single VM
to a multi-site hybrid cloud carrying live traffic. An important
aspect of OnRamp's approach is to describe the internals of the
deployment machinery in enough detail so anyone can deploy and operate
Aether, with each stage highlighting the next set of touchpoints for
customizing the configuration. All of these customizations are
recorded as a set of *blueprints* that govern how Aether is deployed.
(See the ``blueprints`` directory of the OnRamp repo for the currently
available set.)
