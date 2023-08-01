Physical RAN (4G)
----------------------

Aether OnRamp is geared towards 5G, but it does support physical eNBs,
including 4G-based versions of both SD-Core and AMP. It does not,
however, support an emulated 4G RAN. The 4G scenario uses all the same
Ansible machinery outlined in earlier sections, but uses a variant of
``vars/main.yml`` customized for running physical 4G radios:

.. code-block::

   $ cd vars
   $ cp main-eNB.yml main.yml 

Assuming that starting point, the following outlines the key
differences from the 5G case:

1. There is a 4G-specific repo, which you can find in ``deps/4gc``.

2. The ``core`` section of ``vars/main.yml`` specifies a 4G-specific values file:

   ``values_file: "deps/4gc/roles/core/templates/radio-4g-values.yaml"``

3. The ``amp`` section of ``vars/main.yml`` specifies that 4G-specific
   models and dashboards get loaded into the ROC and Monitoring
   services, respectively:

   ``roc_models: "deps/amp/roles/roc-load/templates/roc-4g-models.json"``

   ``monitor_dashboard:  "deps/amp/roles/monitor-load/templates/4g-monitor"``

4. You need to edit two files with details for the 4G SIM cards you
   use. One is the 4G-specific values file used to configure SD-Core:
   
   ``deps/4gc/roles/core/templates/radio-4g-values.yaml``

   The other is the 4G-specific Models file used to bootstrap ROC:
   
   ``deps/amp/roles/roc-load/templates/radio-4g-models.json``
   
5. There are 4G-specific Make targets for SD-Core, including ``make
   aether-4gc-install`` and ``make aether-4gc-uninstall``. Note that
   the generic Make targets for AMP (e.g., ``make
   aether-amp-install`` and ``make aether-amp-uninstall``) work
   unchanged.
   
The Quick Start deployment is for 5G only, but revisiting the Scaling,
Networking, and Physical Radio sections—substituting the above for
their 5G counterparts—serves as a guide for bringing up a 4G version
of Aether.

