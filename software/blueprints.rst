Other Blueprints
-----------------------

The previous sections describe how to deploy four Aether blueprints,
corresponding to four variants of ``var/main.yml``. This section
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
  Emulated RAN section. Minimally, SD-Core runs on one server and
  gNBsim runs on a second server. (The Quick Start deployment, with
  both SD-Core and gNBsim running in the same server, also works.)

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


