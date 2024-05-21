Development Support
-----------------------

OnRamp's primary goal is to support users that want to deploy
officially released versions of Aether on local hardware, but it also
provides a way for users that want to develop new features to deploy
and test them. To this end, this section describes how to configure
OnRamp to use locally modified components, such as Helm Charts and
Docker images (including new images built from source code).

At a low level, development is a component-specific task, and users
are referred to documentation for the respective subsystems:

* To develop SD-Core, see the `SD-Core Guide <https://docs.sd-core.opennetworking.org>`__.

* To develop SD-RAN, see the `SD-RAN Guide <https://docs.sd-ran.org>`__.

* To develop the ROC-based API, see `ROC Development <https://docs.aetherproject.org/master/developer/roc.html>`__.

* To develop Monitoring Dashboards, see `Monitoring Development <https://docs.aetherproject.org/master/developer/monitoring.html>`__.

At a high level, OnRamp provides a means to deploy developmental
versions of Aether that include local modifications to the standard
components. These modifications range from coarse-grain (i.e.,
replacing the Helm Chart for an entire subsystem), to fine-grain
(i.e., replacing the container image for an individual microservice).
The following uses SD-Core as a specific example to illustrate how
this is done. The same approach can be applied to other subsystems.

Local Helm Charts
~~~~~~~~~~~~~~~~~~~~

To substitute a local Helm Chart—for example, one located in directory
``/home/ubuntu/aether/sdcore-helm-charts/sdcore-helm-charts`` on the
server where you run the OnRamp ``make`` targets—edit the ``helm``
block of the ``core`` section of ``vars/main.yml`` to replace:

.. code-block::

  helm:
    local_charts: false
    chart_ref: aether/sd-core
    chart_version: 0.12.8

with

.. code-block::

  helm:
    local_charts: true
    chart_ref: "/home/ubuntu/aether/sdcore-helm-charts/sdcore-helm-charts"
    chart_version: 0.13.2

Note that variable ``core.helm.local_charts`` is a boolean, not the
string ``"true"``. And in this example, we have declared our new chart
to be version ``0.13.2`` instead of ``0.12.8``.

Finally, while there are situations that require modifying full Helm
charts, it is also possibly to simply substitute an alternative values
override file for an existing chart by changing the ``core.values_file:``
variable in ``vars/main.yml``.

Local Container Images
~~~~~~~~~~~~~~~~~~~~~~~~~

Being able to modify a Helm Chart makes it possible to substitute
alternative container images for any or all the microservices
identified in the chart. But it is also possible to substitute just a
single container image while using the standard chart.

To substitute a locally built container image, edit the corresponding
block in the values override file that you have configured in
``vars/main.yml``; e.g.,
``deps/5gc/roles/core/templates/sdcore-5g-values.yaml``.  For example,
if you want to deploy the AMF image with tag ``my-amf:version-foo``
from the container registry of your personal GitLab account, then set
the ``images`` block of 5G control plane section accordingly:

.. code-block::

  5g-control-plane:
    enable5G: true
    images:
      repository: "registry.gitlab.com"
      tags:
        amf: my-account/my-amf:version-foo

A new Make target streamlines the process of frequently re-installing
the Kubernetes pods that implement the Core:

.. code-block::

  $ make 5gc-core-reset

If you are also modifying gNBsim in concert with changes to SD-Core,
then note that the former is not deployed on Kubernetes, and so there
is no Helm Chart or values override file. Instead, you simply need to
modify the ``image`` variable in the ``gnbsim`` section of
``vars/main.yml`` to reference your locally built image:

.. code-block::

  gnbsim:
    docker:
      container:
        image: omecproject/5gc-gnbsim:main-PR_88-cc0d21b

For convenience, the following Make target restarts the container,
which pulls in the new image.

.. code-block::

  $ make gnbsim-reset

Keep in mind that you can also rerun gNBsim with the *same* container,
but loading the latest gNBsim config file, by typing:

.. code-block::

  $ make aether-gnbsim-run

Directly Invoking Helm
~~~~~~~~~~~~~~~~~~~~~~~~~~

It is also possible to directly invoke Helm without engaging OnRamp's
Ansible playbooks. In this scenario, a developer might use OnRamp to
initially set up Aether (e.g., to deploy Kubernetes on a set of nodes,
install the routes and virtual bridges needed to interconnect the
components, and bring up an initial set of pods), but then iteratively
update the pods running on that cluster by executing ``helm``.  This
can be the basis for an efficient development loop for users with an
in-depth understanding of Helm and Kubernetes.

To see how this might work, it is helpful to look at an example
installation playbook, and see how key tasks map onto a corresponding
``helm`` commands. We'll use
``deps/5gc/roles/core/tasks/install.yml``, which installs the 5G core,
as an example. Consider the following two blocks from the playbook
(each block corresponds to an Ansible task):

.. code-block::

  - name: add aether chart repo
    kubernetes.core.helm_repository:
      name: aether
      repo_url: "https://charts.aetherproject.org"
    when: inventory_hostname in groups['master_nodes']

  - name: deploy aether 5gc
    kubernetes.core.helm:
      update_repo_cache: true
      name: sd-core
      release_namespace: omec
      create_namespace: true
      chart_ref: "{{ core.helm.chart_ref }}"
      chart_version: "{{ core.helm.chart_version }}"
      values_files:
        - /tmp/sdcore-5g-values.yaml
      wait: true
      wait_timeout: "2m30s"
      force: true
    when: inventory_hostname in groups['master_nodes']

These two tasks correspond to the following three ``helm`` commands:

.. code-block::

   $ helm repo add aether https://charts.aetherproject.org
   $ helm repo update
   $ helm upgrade --create-namespace \
                            --install \
                            --version $CHART_VERSION \
                            --wait \
                            --namespace omec \
                            --values $VALUES_FILE \
                            sd-core

The correspondence between task parameters and command arguments is
straightforward, keeping in mind that both approaches take advantage
of variables (as defined in ``vars/main.yml`` for the Ansible tasks,
and corresponding to shell variables ``CHART_VERSION`` and
``VALUES_FILE`` in our example command sequence). The ``when`` line in
the two tasks indicates that the task is to be run on the
``master_nodes`` in your ``hosts.ini`` file; that node is where you
would directly call ``helm``. Note that local charts can be used by
also executing the following command (reusing the example path name
from earlier in this section):

.. code-block::

   $ helm dep up /home/ubuntu/aether/sdcore-helm-charts/sdcore-helm-charts

You will see other tasks in the OnRamp playbooks. These tasks
primarily take care of bookkeeping; automating bookkeeping tasks
(including templating) is one of the main values that Ansible provides.

Finally, keep in mind that in using SD-Core to illustrate how to build
a customized modify-and-test loop, this section doesn't address some
of the peculiarities of the other components. As one example, ROC has
prerequisites that have to be installed before the ROC itself. These
prereqs are identified in the ROC installation playbook, and include
``onos-operator``, which in turn depends on ``atomix``.

As another example, the ROC and monitoring services allow you to
program new features by loading alternative "specifications" into the
running pods (in addition to installing new container images).  This
approach is described in the `ROC Development
<https://docs.aetherproject.org/master/developer/roc.html>`__ and
`Monitoring Development
<https://docs.aetherproject.org/master/developer/monitoring.html>`__
sections, respectively, and implemented by the ``roc-load`` and
``monitor-load`` roles found in ``deps/amp/roles``.





