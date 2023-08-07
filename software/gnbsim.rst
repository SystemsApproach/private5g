Emulated RAN
----------------

gNBsim emulates a 5G RAN, generating (mostly) Control Plane traffic
that can be directed at SD-Core. This section describes how to
configure gNBsim, so as to both customize and scale the workload it
generates. We assume gNBsim runs in one or more servers, independent
of the server(s) that host SD-Core. These servers are specified in the
``hosts.ini`` file, as described in the section on Scaling Aether. We
also assume you start with a variant of ``vars/main.yml`` customized
for running gNBsim, which is easy to do:

.. code-block::

   $ cd vars
   $ cp main-gnbsim.yml main.yml

Configure gNBsim
~~~~~~~~~~~~~~~~~~

Two sets of parameters control gNBsim. The first set, found in the
``gnbsim`` section of ``vars/main.yml``, controls how gNBsim is
deployed: (1) the number of servers it runs on; (2) the number of
Docker containers running within each server; (3) what configuration
to run in each of those containers; and
(4) how those containers connect to SD-Core. For example, consider the
following variable definitions:

.. code-block::

   gnbsim:
       docker:
           container:
               image: omecproject/5gc-gnbsim:main-PR_88-cc0d21b
               prefix: gnbsim
               count: 2
           network:
              macvlan:
                   name: gnbnet

       router:
          data_iface: ens18
          macvlan:
               iface: gnbaccess
               subnet_prefix: "172.20"

       servers:
           0:
              - "config/gnbsim-s1-p1.yaml"
              - "config/gnbsim-s1-p2.yaml"
           1:
              - "config/gnbsim-s2-p1.yaml"
              - "config/gnbsim-s2-p2.yaml"

The ``container.count`` variable in the ``docker`` block specifies how
many containers run in each server (``2`` in this example). The
``router`` block then gives the network specification needed for these
containers to connect to the SD-Core; all of these variables are
described in the previous section on Networking. Finally, the
``servers`` block names the configuration files that parameterize each
container. In this example, there are two servers with two containers
running in each, with ``config/gnbsim-s2-p1.yaml`` parameterizing the
first container on the second server.

These config files then specify the second set of gNBsim parameters.
A detailed description of these parameters is outside the scope of
this guide (see https://github.com/omec-project/gnbsim for details),
but at a high-level, gNBsim defines a set of *profiles*, each of which
exercises a common usage scenario that the Core has to deal with. Each
of these sequences is represented by a ``profileType`` in the config
file. gNBsim supports seven profiles, which we list here:

.. code-block::

   - profileType: register		# UE Registration
   - profileType: pdusessest		# UE Initiated Session
   - profileType: anrelease		# Access Network (AN) Release
   - profileType: uetriggservicereq	# UE Initiated Service Request
   - profileType: deregister		# UE Initiated De-registration
   - profileType: nwtriggeruedereg	# Network Initiated De-registration
   - profileType: uereqpdusessrelease	# UE Initiated Session Release

The second profile (``pdusettest``) is selected by default. It causes
the specified number of UEs to register with the Core, initiate a user
plane session, and then send a minimal data packet over that session.
Note that the rest of the per-profile parameters are highly redundant.
For example, they specify the IMSI- and PLMD-related information UEs
need to connect to the Core.

Finally, it is necessary to edit the ``core`` section of
``vars/main.yml`` to indicate the address at which gNBsim can find the
AMF. For our running example, this would look like the following:

.. code-block::

   core:
       amf: "10.76.28.113"


Install/Uninstall gNBsim
~~~~~~~~~~~~~~~~~~~~~~~~~~

Once you have edited the parameters (and assuming you already have
SD-Core running), you are ready to install gNBsim. This includes starting
up all the containers and configuring the network so they can reach
the Core. This is done from the main OnRamp server you've been using,
where you type:

.. code-block::

   $ make gnbsim-docker-install
   $ make aether-gnbsim-install

Note that the first step may not be necessary, depending on whether
Docker is already installed on the server(s) you've designated to host
gNBsim.

When you are finished, the following uninstalls everything:

.. code-block::

   $ make aether-gnbsim-uninstall

Run gNBsim
~~~~~~~~~~~~~~~~~~

Once gNBsim is installed and the Docker containers instantiated, you
can run the simulation by typing

.. code-block::

   $ make aether-gnbsim-run

This can be done multiple times without reinstalling. For each run,
you can use Docker to view the results, which have been saved in each
of the containers. To do so, ssh into one of the servers designated to
to run gNBsim, and then type

.. code-block::

   $ docker exec -it gnbsim-1 cat summary.log

Note that container name ``gnbsim-1`` is constructed from the
``prefix`` variable defined in the ``docker`` section of
``vars/main.yml``, with ``-1`` indicating the first container.
