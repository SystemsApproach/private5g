Other Blueprints
-----------------------

The previous sections describe how to deploy three Aether blueprints,
corresponding to three variants of ``var/main.yml``. This section
documents additional blueprints, each defined by a combination of
Ansible components:

* A ``vars/main-blueprint.yml`` file, checked into the
  ``aether-onramp`` repo, is the "root" of the blueprint
  specification.

* A ``hosts.ini`` file, documented by example, specifies the target
  servers required by the blueprint.

* A set of Make targets, defined in a submodule and imported into
  OnRamp's global Makefile, provides commands to install and uninstall
  the blueprint.

* (Optional) A new ``aether-blueprint`` repo defines the Ansible Roles
  and Playbooks required to deploy a new component.

* (Optional) New Roles, Playbooks, and Templates, checked to existing
  repos/submodules, customize existing components for integration with
  the new blueprint. To support blueprint independence, these elements
  are intentionally kept "narrow", rather than glommed onto an
  existing element.

* (Optional) Any additional hardware (beyond the Ansible-managed
  Aether servers) required to support the blueprint.

* A Jenkins job, added to the set of OnRamp integration tests,
  verifies that the blueprint successfully deploys Aether.

The goal of establishing a well-defined procedure for adding new
blueprints to OnRamp is to encourage the community to contribute (and
maintain) new Aether configurations and deployment scenarios.\ [#]_
The rest of this section documents community-contributed blueprints
to-date.

.. [#] Not all possible configurations of Aether require a
       blueprint. There are other ways to add variability, for
       example, by documenting simple ways to modify an existing
       blueprint.  Disabling ``core.standalone`` and selecting an
       alternative ``core.values_file`` are two common examples.

Multiple UPFs
~~~~~~~~~~~~~~~~~~~~~~

The base version of SD-Core includes a single UPF, running in the same
Kubernetes namespace as the Core's control plane. This blueprint adds
the ability to bring up multiple UPFs (each in a different namespace),
and uses ROC to establish the *UPF-to-Slice-to-Device* bindings
required to activate end-to-end user traffic. The resulting deployment
is then verified using gNBsim.

The Multi-UPF blueprint includes the following:

* Global vars file ``vars/main-upf.yml`` gives the overall
  blueprint specification.

* Inventory file ``hosts.ini`` is identical to that used in the
  `Emulated RAN <gnbsim.html>`__ section.  Minimally,
  SD-Core runs on one server and gNBsim runs on a second server.
  (The Quick Start deployment, with both SD-Core and gNBsim running
  in the same server, also works.)

* New make targets, ``5gc-upf-install`` and ``5gc-upf-uninstall``, to
  be executed after the standard SD-Core installation. The blueprint
  also reuses the ``roc-load`` target to activate new slices in ROC.

* New Ansible role (``upf``) added to the ``5gc`` submodule, including
  a new UPF-specific template (``upf-5g-values.yaml``).

* New models file (``roc-5g-models-upf2.json``) added to the
  ``roc-load`` role in the ``amp`` submodule. This models file is
  applied as a patch *on top of* the base set of ROC models. (Since
  this blueprint is demonstrated using gNBsim, the assumed base models
  are given by ``roc-5g-models.json``.)

* Two nightly integration tests that validate the Multi-UPF blueprint
  can be viewed on Jenkins (assuming you are a registered user):
  `single-server test
  <https://jenkins.aetherproject.org/view/Aether%20OnRamp/job/AetherOnRamp_QuickStart_Multi-UPF/>`__,
  `two-server test
  <https://jenkins.aetherproject.org/view/Aether%20OnRamp/job/AetherOnRamp_2servers_Multi-UPF/>`__.

To use Multi-UPF, first copy the vars file to ``main.yml``:

.. code-block::

   $ cd vars
   $ cp main-upf.yml main.yml

Then edit ``hosts.ini`` and ``vars/main.yml`` to match your local
target servers, and deploy the base system (as in previous sections):

.. code-block::

   $ make k8s-install
   $ make roc-install
   $ make roc-load
   $ make 5gc-core-install
   $ make gnbsim-install

You can also optionally install the monitoring subsystem. Note that
because ``main.yml`` sets ``core.standalone: "false"``, any models
loaded into ROC are automatically applied to SD-Core.

At this point you are ready to bring up additional UPFs and bind them
to specific slices and devices. This involves first editing the
``upf`` block in the ``core`` section of ``vars/main.yml``:

.. code-block::

   upf:
      ip_prefix: "192.168.252.0/24"
      iface: "access"
      helm:
          chart_ref: aether/bess-upf
     values_file: "deps/5gc/roles/upf/templates/upf-5g-values.yaml"
     additional_upfs:
         "1":
            ip:
               access: "192.168.252.6/24"
               core:   "192.168.250.6/24"
            ue_ip_pool: "172.248.0.0/16"
         # "2":
         #   ip:
         #      access: "192.168.252.7/24"
         #      core:   "192.168.250.7/24"
         #   ue_ip_pool: "172.247.0.0/16"

As shown above, one additional UPF is enabled (beyond ``upf-0`` that
already came up as part of SD-Core), with the spec for yet another UPF
commented out.  In this example configuration, each UPF is assigned a
subnet on the ``access`` and ``core`` bridges, along with the IP
address pool for UEs that the UPF serves.  Once done with the edits,
launch the new UPF(s) by typing:

.. code-block::

   $ make 5gc-upf-install

At this point the new UPF(s) will be running (you can verify this
using ``kubectl``), but no traffic will be directed to them until UEs
are assigned to their IP address pool. Doing so requires loading the
appropriate bindings into ROC, which you can do by editing the
``roc_models`` line in ``amp`` section of ``vars/main.yml``. Comment
out the original models file already loaded into ROC, and uncomment
the new patch that is to be applied:

.. code-block::

   amp:
      # roc_models: "deps/amp/roles/roc-load/templates/roc-5g-models.json"
      roc_models: "deps/amp/roles/roc-load/templates/roc-5g-models-upf2.json"

Then run the following to load the patch:

.. code-block::

   $ make roc-load

At this point you can bring up the Aether GUI and see that a second
slice and a second device group have been mapped onto the second UPF.

Now you are ready to run traffic through both UPFs, which because the
configuration files identified in the ``servers`` block of the
``gnbsim`` section of ``vars/main.yml`` align with the IMSIs bound to
each Device Group (which are bound to each slice, which are in turn
bound to each UPF), the emulator sends data through both UPFs.  To run
the emulation, type:

.. code-block::

   $ make gnbsim-simulator-run

SD-RAN
~~~~~~~~~~~~~~~~~~~~~~

This blueprint runs SD-Core and SD-RAN in tandem, with RANSIM
emulating various RAN elements. (The OnRamp roadmap includes plans to
couple SD-RAN with other virtual and physical RAN elements, but RANSIM
is currently the only option.)

The SD-RAN blueprint includes the following:

* Global vars file ``vars/main-sdran.yml`` gives the overall
  blueprint specification.

* Inventory file ``hosts.ini`` is identical to that used in the Quick
  Start deployment, with both SD-RAN and SD-Core co-located on a
  single server.

* New make targets, ``aether-sdran-install`` and
  ``aether-sdran-uninstall``, to be executed after the standard
  SD-Core installation.

* A new submodule ``deps/sdran`` (corresponding to repo
  ``aether-sdran``) defines the Ansible Roles and Playbooks required
  to deploy SD-RAN.

* A nightly integration test that validates the SD-RAN blueprint can
  be viewed on `Jenkins
  <https://jenkins.aetherproject.org/view/Aether%20OnRamp/job/AetherOnRamp_QuickStart_SDRAN/>`__
  (assuming you are a registered user).

To use SD-RAN, first copy the vars file to ``main.yml``:

.. code-block::

   $ cd vars
   $ cp main-sdran.yml main.yml

Then edit ``hosts.ini`` and ``vars/main.yml`` to match your local
target servers, and deploy the base system (as in previous sections),
followed by SD-RAN:

.. code-block::

   $ make aether-k8s-install
   $ make aether-5gc-install
   $ make aether-sdran-install

Use ``kubectl`` to validate that the SD-RAN workload is running, which
should result in output similar to the following:

.. code-block::

   $ kubectl get pods -n sdran
   NAME                             READY   STATUS    RESTARTS   AGE
   onos-a1t-68c59fb46-8mnng         2/2     Running   0          3m12s
   onos-cli-c7d5b54b4-cddhr         1/1     Running   0          3m12s
   onos-config-5786dbc85c-rffv7     3/3     Running   0          3m12s
   onos-e2t-5798f554b7-jgv27        2/2     Running   0          3m12s
   onos-kpimon-555c9fdb5c-cgl5b     2/2     Running   0          3m12s
   onos-topo-6b59c97579-pf5fm       2/2     Running   0          3m12s
   onos-uenib-6f65dc66b4-b78zp      2/2     Running   0          3m12s
   ran-simulator-5d9465df55-p8b9z   1/1     Running   0          3m12s
   sd-ran-consensus-0               1/1     Running   0          3m12s
   sd-ran-consensus-1               1/1     Running   0          3m12s
   sd-ran-consensus-2               1/1     Running   0          3m12s

Note that the SD-RAN workload includes RANSIM as one of its pods;
there is no separate "run simulator" step as is the case with gNBsim.
To validate that the emulation ran correctly, query the ONOS CLI as
follows:

Check ``onos-kpimon`` to see if 6 cells are present:

.. code-block::

   $ kubectl exec -it deployment/onos-cli -n sdran -- onos kpimon list metrics

Check ``ran-simulator`` to see if 10 UEs and 6 cells are present:

.. code-block::

   $ kubectl exec -it deployment/onos-cli -n sdran -- onos ransim get cells
   $ kubectl exec -it deployment/onos-cli -n sdran -- onos ransim get ues

Check ``onos-topo`` to see if ``E2Cell`` is present:

.. code-block::

   $ kubectl exec -it deployment/onos-cli-n sdran -- onos topo get entity -v

UERANSIM
~~~~~~~~~~~~~~~~~~~~~~

This blueprint runs UERANSIM in place of gNBsim, providing a second
way to direct workload at SD-Core. Of particular note, UERANSIM runs
``iperf3``, making it possible to measure UPF throughput. (In
contrast, gNBsim primarily stresses the Core's Control Plane.)

The UERANSIM blueprint includes the following:

* Global vars file ``vars/main-ueransim.yml`` gives the overall
  blueprint specification.

* Inventory file ``hosts.ini`` needs to be modified to identify the
  server that is to run UERANSIM. Currently, a second server is
  needed, as UERANSIM and SD-Core cannot be deployed on the same
  server. As an example, ``hosts.ini`` might look like this:

.. code-block::

   [all]
   node1  ansible_host=10.76.28.113 ansible_user=aether ansible_password=aether ansible_sudo_pass=aether
   node2  ansible_host=10.76.28.115 ansible_user=aether ansible_password=aether ansible_sudo_pass=aether

   [master_nodes]
   node1

   [ueransim_nodes]
   node2

* New make targets, ``aether-ueransim-install``,
  ``aether-ueransim-run``, and ``aether-ueransim-uninstall``, to be
  executed after the standard SD-Core installation.

* A new submodule ``deps/ueransim`` (corresponding to repo
  ``aether-ueransim``) defines the Ansible Roles and Playbooks
  required to deploy UERANSIM. It also contains configuration files
  for the emulator.

* A nightly integration test that validate the UERANSIM blueprint
  can be viewed on Jenkins (assuming you are a registered user):
  `two-server test
  <https://jenkins.aetherproject.org/view/Aether%20OnRamp/job/AetherOnRamp_2servers_20.04_default-charts_UERANSIM/>`__.


To use UERANSIM, first copy the vars file to ``main.yml``:

.. code-block::

   $ cd vars
   $ cp main-ueransim.yml main.yml

Then edit ``hosts.ini`` and ``vars/main.yml`` to match your local
target servers, and deploy the base system (as in previous sections),
followed by UERANSIM:

.. code-block::

   $ make aether-k8s-install
   $ make aether-5gc-install
   $ make aether-ueransim-install
   $ make aether-ueransim-run

The last step actually starts UERANSIM, configured according to the
specification given in files ``custom-gnb.yaml`` and
``custom-ue.yaml`` located in ``deps/ueransim/config``. Make target
``aether-ueransim-run`` can be run multiple times, where doing so
reflects any recent edits to the config files. More information about
UERANSIM can be found on `GitHub
<https://github.com/aligungr/UERANSIM>`__, including how to set up the
config files.

Finally, since the main value of UERANSIM is to measure user plane
throughput, you may want to play with the UPF's Quality-of-Service
parameters, as defined in
``deps/5gc/roles/core/templates/sdcore-5g-values.yaml``. Specifically,
see both the UE-level settings associated with ``ue-dnn-qos`` and the
slice-level settings associated with ``slice_rate_limit_config``.

Physical eNBs
~~~~~~~~~~~~~~~~~~

Aether OnRamp is geared towards 5G, but it does support physical eNBs,
including 4G-based versions of both SD-Core and AMP.  The 4G blueprint
has been demonstrated with `SERCOMM's 4G/LTE CBRS Small Cell
<https://wiki.aetherproject.org/display/HOME/Certified+Hardware>`__.
The blueprint uses all the same Ansible machinery outlined in earlier
sections, but starts with a variant of ``vars/main.yml`` customized
for running physical 4G radios:

.. code-block::

   $ cd vars
   $ cp main-eNB.yml main.yml

Assuming that starting point, the following outlines the key
differences from the 5G case:

* There is a 4G-specific repo, which you can find in ``deps/4gc``.

* The ``core`` section of ``vars/main.yml`` specifies a 4G-specific values file:

  ``values_file: "deps/4gc/roles/core/templates/radio-4g-values.yaml"``

* The ``amp`` section of ``vars/main.yml`` specifies that 4G-specific
  models and dashboards get loaded into the ROC and Monitoring
  services, respectively:

  ``roc_models: "deps/amp/roles/roc-load/templates/roc-4g-models.json"``

  ``monitor_dashboard:  "deps/amp/roles/monitor-load/templates/4g-monitor"``

* You need to edit two files with details for the 4G SIM cards you
  use. One is the 4G-specific values file used to configure SD-Core:

  ``deps/4gc/roles/core/templates/radio-4g-values.yaml``

  The other is the 4G-specific Models file used to bootstrap ROC:

  ``deps/amp/roles/roc-load/templates/radio-4g-models.json``

* There are 4G-specific Make targets for SD-Core (e.g., ``make
  aether-4gc-install`` and ``make aether-4gc-uninstall``), but the
  Make targets for AMP (e.g., ``make aether-amp-install`` and ``make
  aether-amp-uninstall``) work unchanged in both 4G and 5G.

The Quick Start and Emulated RAN (gNBsim) deployments are for 5G only,
but revisiting the previous sections—substituting the above for their
5G counterparts—serves as a guide for deploying a 4G blueprint of
Aether.  Note that the network is configured in exactly the same way
for both 4G and 5G. This is because SD-Core's implementation of the
UPF is used in both cases.
