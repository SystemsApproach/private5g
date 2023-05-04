Stage 4: Runtime Control
--------------------------

Aether defines an API (and associated GUI) for managing connectivity
at runtime. This stage brings up that API/GUI, as implemented by the
*Runtime Operational Control (ROC)* subsystem, building on the
physical gNB we connected to Aether in Stage 3. As in the previous
stage, we focus on the ``5g-radio`` blueprint, although the same
approach applies equally well to the ``4g-radio`` and ``latest``
blueprints.

This stage focuses on the abstractions that the ROC layers on top of
the SD-Core. These abstractions are described in `Section 6.4
<https://5g.systemsapproach.org/cloud.html#connectivity-api>`__ and
include *Device Groups* and *Slices*. Initial settings of these
ROC-managed parameters are recorded in
``5g-radio/roc-5g-models.json``. We use these values to bootstrap the
ROC database, saving us from a laborious GUI session.

Somewhat confusingly, the *Device-Group* and *Slice* information is
duplicated between ``5g-radio/sd-core-5g-values.yaml`` and
``5g-radio/roc-5g-models.json``. This makes it possible to bring up
the SD-Core without the ROC, which simplifies the process of debugging
an initial installation (as was the case in Stage 3), but having two
sources for this information leads to problems keeping them in sync,
and should be avoided.

Aether is designed to treat the ROC as the "single source of truth"
for *Slices* and *Device Groups* (along with the other abstract
objects it defines), so we recommend using the GUI or API to make
changes over time, and avoiding the override values once you've
established basic connectivity. And if you want to save this
"bootstrap state" in a text file for a possible restart, we recommend
doing so in ``5g-radio/roc-5g-models.json``.

To adopt this approach, first edit the ``SA_CORE`` variable in
``blueprints/5g-radio/config``, setting it to ``false``. This variable
indicates whether we want SD-Core to run in *Stand Alone* mode, which
has been the default setting up to this point. Specifically, disabling
``SA_CORE`` causes the SD-Core to ignore the ``device-groups`` and
``network-slices`` blocks of the ``omec-sub-provision`` section in
``5g-radio/sd-core-5g-values.yaml``, and to instead retrieve this
information from the ROC.

The next step is to edit ``5g-radio/roc-5g-models.json`` to record the
same IMSI information you added to
``5g-radio/sd-core-5g-values.yaml`` in Stage 3.  Do this by modifying,
adding and removing ``sim-card`` entries as necessary. Note that only
the IMSIs need to match the earlier data; the ``sim-id`` and
``display-name`` values are arbitrary and need only be consistent
*within* ``5g-radio/roc-5g-models.json``.

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
running in Stage 3, and then bring up the ROC followed by a new
instantiation of the SD-Core:

.. code-block::

   $ make core-clean
   $ make 5g-roc
   $ make 5g-core

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
values, and then click ``Update`` to save them.

Similarly, if you select ``Sim Cards`` from the drop-down menu at top
right, the ``Edit`` icon associated with each SIM card allows you to
see (and potentially change) the IMSI values associated with each
device.  You can also disable individual IMSIs. Again, click
``Update`` if you make any changes.

The registered IMISs can be aggregated into *Device-Groups* by
selecting ``Device-Groups`` from the drop-down menu at the top right,
and adding a new device group.

Finally, if you do make a set of edits, select the ``Basket`` icon at
top right when you are done, and click the ``Commit`` button. This
causes the set of changes to be committed as a single transaction.
