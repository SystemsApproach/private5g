Scale Aether
-----------------

Everything up to this point has been done as part of the Quick Start
tutorial, with all the components running in a single host (VM or
physical machine). We now describe how to scale Aether to run on
multiple hosts, with this cluster-based configuration used throughout
the rest of this Appendix. Before continuing, though, you will need to
remove the Quick Start configuration, by typing:

.. code-block::

   root@host:/workdir# make aether-uninstall

There are two aspects of our deployment that scale independently. One
is Aether proper: a Kubernetes cluster running the set of microsevices
that implement SD-Core and AMP. The other is gNBsim: the emulated RAN
that generates traffic directed at the Aether cluster. Minimally, two
hosts are required—one for the Aether cluster and one for gNBsim—with
each able to grow independently. For example, having four hosts would
support a 3-node Aether cluster and a 1-node workload generator. This
is the configuration shown in ``hosts.ini`` assuming you uncomment all
the lines:

.. code-block::

   [all]
   node1 ansible_host=172.16.144.50 ansible_user=aether ansible_password=aether ansible_sudo_pass=aether
   node2 ansible_host=172.16.144.71 ansible_user=aether ansible_password=aether ansible_sudo_pass=aether
   node3 ansible_host=172.16.144.18 ansible_user=aether ansible_password=aether ansible_sudo_pass=aether
   node4 ansible_host=172.16.144.93 ansible_user=aether ansible_password=aether ansible_sudo_pass=aether

   [master_nodes]
   node1

   [worker_nodes]
   node2
   node3
   node4

   [aether_nodes]
   node1
   node2
   node3
   
   [gnbsim_nodes]
   node4

The first block identifies all the nodes; the second block designates
which node runs the Ansible client (this is the node you ssh into and
invoke the Make targets); the third block designates the worker nodes
being managed by the Ansible client; and the last two blocks indicate
which nodes constitue the Aether cluster (Kubernetes is installed on
these nodes) versus runs the gNBsim workload generator (gNBsim scales
across mulitple Docker containers, outside Kubernetes).

In addition to modifying ``hosts.ini`` to match your deployment
target, you need to change one parameter in in ``vars/main.yml``,
setting ``SameMachineAsCore`` to false in the ``gnbsim`` section:

.. code-block::

   gnbsim:
     SameMachineAsCore: false

Once you've done that (and assuming you deleted your earlier Quick
Start configuration), you can reexecute the same set of targets you
ran before:

.. code-block::

   root@host:/workdir# make k8s-install
   root@host:/workdir# make 5g-core-install
   root@host:/workdir# make amp-install
   root@host:/workdir# make gnbsim-install
   root@host:/workdir# make gnbsim-run

This will run the same gNBsim test case as before. We will return to
options for scaling up the gNBsim workload in a later section, as well
as describe how to run physical gNBs in place of gNBsim. Note that if
you are primarily interested in the latter, you can still run Aether
on a single node, and then connect that node to a physical gNB; a
multi-node cluster is not require.
