Physical RAN (4G)
----------------------

Aether OnRamp is geared towards 5G, but it does support physical eNBs,
including 4G-based versions of both SD-Core and AMP. It does not,
however, support an emulated 4G RAN. The 4G scenario uses all the same
Ansible machinery outlined in earlier sections, changing only the
following details:

1. There is a 4G-specific repo, which you can find in ``deps/4gc``.

2. You need to edit the ``core`` section of ``vars/main.yml`` to
   specify a 4G-specific values file:

   ``values_file: "deps/4gc/templates/core/radio-4g-values.yaml"``

3. You need to edit two files with details for the 4G SIM cards you
   use. One is the 4G-specific values file used to configure SD-Core:
   
   ``deps/4gc/templates/core/radio-4g-values.yaml``

   The other is the 4G-specific Models file used to bootstrap ROC:
   
   ``deps/amp/roles/4g-roc/templates/roc-4g-models.json``

4. There are 4G-specific Make targets for SD-Core, including ``make
   aether-4gc-install`` and ``make aether-4gc-uninstall``.

5. There are 4G-specific Make targets for AMP, including ``make
   aether-4g-amp-install`` and ``make aether-4g-amp-uninstall``.

The Quick Start deployment is 5G only, but revisiting the Scaling,
Networking, and Physical Radio sections—substituting the above for
their 5G counterparts—serves as a guide for bringing up a 4G version
of Aether.
   
