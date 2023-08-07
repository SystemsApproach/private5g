Runtime Control
-----------------------------------

Aether defines an API (and associated GUI) for managing connectivity
at runtime. This stage brings up that API/GUI, as implemented by the
*Runtime Operational Control (ROC)* subsystem, building on the
physical gNB we connected to Aether in the previous section.

This stage focuses on the abstractions that the ROC layers on top of
the SD-Core. These abstractions are described in `Section 6.4
<https://5g.systemsapproach.org/cloud.html#connectivity-api>`__ and
include *Device Groups* and *Slices*. (The full set of model
definitions can be found in `GitHub
<https://github.com/onosproject/aether-models>`__.)  Initial settings
of these ROC-managed parameters are recorded in
``deps/amp/roles/roc-load/templates/radio-5g-models.json``. We use these
values to load the ROC database, saving us from a laborious GUI
session.

Somewhat confusingly, the *Device-Group* and *Slice* information is
duplicated between ``deps/5gc/roles/core/templates/radio-5g-values.yaml``
and this ``radio-5g-models.json`` file. This makes it possible to bring
up the SD-Core without the ROC, which simplifies the process of
debugging an initial installation, but having two sources for this
information leads to problems keeping them in sync, and should be
avoided.

To this end, Aether treats the ROC as the "single source of truth" for
*Slices*, *Device Groups*, and all the other abstract objects it
defines, so we recommend using the GUI or API to make changes over
time, and avoiding the override values in ``radio-5gc-values.yaml``
once you've established basic connectivity. And if you want to save
this bootstrap state in a text file for a possible restart, we
recommend doing so in ``radio-5g-models.json`` (although this is not a
substitute for the operational practice of backing up the ROC
database).

To make ROC the authoritative source of runtime state, first edit the
``standalone`` variable in the ``core`` section of ``vars/main.yml``,
setting it to ``false``. This variable indicates whether we want
SD-Core to run in *Stand Alone* mode, which has been the default
setting up to this point. Disabling ``standalone`` causes the SD-Core
to ignore the ``device-groups`` and ``network-slices`` blocks of the
``omec-sub-provision`` section in ``radio-5gc-values.yaml``, and to instead
retrieve this information from the ROC.

The next step is to edit ``radio-5g-models.json`` to record the same
IMSI information you added to ``radio-5gc-values.yaml`` in the
previous section.  This includes modifying, adding and removing
``sim-card`` entries as necessary. Note that only the IMSIs need to
match the earlier data; the ``sim-id`` and ``display-name`` values are
arbitrary and need only be consistent *within* ``radio-5g-models.json``.

.. code-block::

   "imsi-definition": {
          "mcc": "315",
          "mnc": "010",
          "enterprise": 1,
          "format": "CCCNNNEESSSSSSS"
   },
   ...

   "sim-card": [
          {
              "sim-id": "sim-1",
              "display-name": "SIM 1",
              "imsi": "315010999912301"
          },
   ...

Once you are done with these edits, uninstall the SD-Core you had
running in the previous stage, and then bring up the ROC followed by a
new instantiation of the SD-Core:

.. code-block::

   $ make aether-5gc-uninstall
   $ make aether-amp-install
   $ make aether-5gc-install

The order is important, since the Core depends on configuration
parameters provided by the ROC. Also note that you may need to reboot
the gNB, although it typically does so automatically when it detects
that the Core has restarted.

To see these initial configuration values using the GUI, open the
dashboard available at ``http://<server-ip>:31194``. If you select
``Configuration > Site`` from the drop-down menu at top right, and
click the ``Edit`` icon associated with the ``Aether Site`` you can
see (and potentially change) the following values:

* MCC: 315
* MNC: 010

Although we have no need to do so now, you can make changes to these
values, and then click ``Update`` to save them to the "commit basket".
Similarly, if you select ``Sim Cards`` from the drop-down menu at top
right, the ``Edit`` icon associated with each SIM card allows you to
see (and potentially change) the IMSI values associated with each device.
You can also disable individual IMSIs. Again, click ``Update`` if you
make any changes.

The set of registered IMISs can be aggregated into *Device-Groups* by
selecting ``Device-Groups`` from the drop-down menu at the top right,
and adding a new device group.

Finally, if you do make a set of updates, select the ``Basket`` icon
at top right when you are done, and click the ``Commit`` button. This
causes the set of changes to be committed as a single transaction.

A more complete User's Guide for the ROC is available online, although
be aware that our OnRamp-based deployment has not yet enabled the
secure login feature.

.. _reading_roc:
.. admonition:: Further Reading

    `Aether Operations
    <https://docs.aetherproject.org/master/operations/gui.html>`__.
