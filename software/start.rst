Quick Start
-----------------------

Aether OnRamp provides a low-overhead way to get started. It brings up
a one-node Kubernetes cluster, deploys a 5G version of SD-Core on that
cluster, and runs an emulated 5G workload against the 5G Core. It
assumes a low-end server that meets the following requirements:

* Haswell CPU (or newer), with at least 4 CPUs and 12GB RAM.
* Clean install of Ubuntu 18.04, 20.04, or 22.04, with 4.15 (or later) kernel.

For example, something like an Intel NUC is more than enough to get
started.

While this appendix focuses on deploying Aether OnRamp on a physical
machine (in anticipation of later stages), this stage can also run in
a VM.  Options include an AWS VM (Ubuntu 20.04 image on `t2.xlarge`
instance); a VirtualBox VM running `bento/ubuntu-20.04` `Vagrant
<https://www.vagrantup.com>`_ box on Intel Mac; a VM created using
`Multipass <https://multipass.run>`_ on Linux, Mac, or Windows; or
`VMware Fusion <https://www.vmware.com/products/fusion.html>`__ to run
a VM on a Mac.

For example, if you have Multipass installed on your laptop, you can
launch a suitable VM instance by typing:

.. code-block::

   $ multipass launch 20.04 --cpus 4 --disk 50G --memory 16G --name onramp

Prep Environment
~~~~~~~~~~~~~~~~~~~~~

To install Aether OnRamp, you must be able able to run `sudo` without
a password, and there should be no firewall running on the server,
which you can verify as follows:

.. code-block::

   $ sudo ufw status
   $ sudo iptables -L
   $ sudo nft list

The first command should report inactive, and the second two commands
should return blank configurations.

Because the install process fetches artifacts from the Internet, if you
are behind a proxy you will need to set the standard Linux environment
variables: `http_proxy`, `https_proxy`, `no_proxy`, `HTTP_PROXY`,
`HTTPS_PROXY` and `NO_PROXY` with the appropriate values. You also
need to export `PROXY_ENABLED=true` by typing the following:

.. code-block::

   $ export PROXY_ENABLED=true

This variable can also be set in your ``~/.bashrc`` file to make it
permanent.

Proxy or no-proxy, using OnRamp involves downloading many Docker
images and Helm Charts. If any of the steps described below fail, it
may be due to this download taking too long, in which case
re-executing the step is usually all it takes to make progress.

Download Aether OnRamp
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once ready, clone the Aether OnRamp repo on this target deployment
server:

.. code-block::

   $ mkdir ~/systemsapproach
   $ cd ~/systemsapproach
   $ git clone --recursive https://github.com/opennetworkinglab/aether-onramp.git
   $ cd aether-onramp

Taking a quick look at your ``aether-onramp`` directory, there are
four things to note:

1. The ``deps`` directory contains the Ansible deployment
   specifications for all the Aether subsystems. Each of these
   subdirectories (e.g., ``deps/5gc``) is self-contained, meaning you
   can execute the Make targets in each individual directory. Doing so
   causes Ansible to run the corresponding playbook. For example, the
   installation playbook for the 5G Core can be found in
   ``deps/5gc/roles/core/tasks/install.yml``.

2. The Makefile in the main OnRamp directory imports (``#include``)
   the per-subsystem Makefiles, meaning all the individual steps
   required to install Aether can be managed from this main directory.
   The Makefile includes comments listing the key Make targets defined
   by the included Makefiles. *Importantly, the rest of this Appendix
   assumes you are working in the main OnRamp directory, and not in
   the individual subsystems.*

3. File ``vars/main.yml`` defines all the Ansible variables you will
   potentially need to modify to specify your deployment scenario.
   This file is the union of all the per-component ``var/main.yml``
   files you find in the corresponding ``deps`` directory. This
   top-level variable file overrides the per-component var files, so
   you will not need to modify the latter. *Importantly, be aware that
   some variables (e.g.,* ``data_iface``\ *) show up in multiple sections
   of this top-level var file.*

4. File ``hosts.ini`` (host inventory) is Ansible's way of specifying
   the set of servers (physical or virtual) that Ansible targets with
   various installation playbooks. The default version of ``host.ini``
   included with OnRamp is simplified to run everything on a single
   server (the one you've cloned the repo onto), with additional lines
   you may eventually need for a multi-node cluster commented out.
 

Set Target Parameters
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Quick Start tutorial described in this section requires that you
modify two parameters to reflect the specifics of your target
deployment.

The first is in file ``host.ini``, where you will need to give the IP
address and login credentials for the server you are working on. At
this stage, we assume the server you downloaded OnRamp onto is the
same server you will be installing Aether on.

.. code-block::

   node1  ansible_host=172.16.41.103 ansible_user=aether ansible_password=aether ansible_sudo_pass=aether

In this example, address ``172.16.41.103`` and the three occurrences of
the string ``aether`` need to be replaced with the appropriate values.
Note that if you set up your server to use SSH keys instead of
passwords, then ``ansible_password=aether`` needs to be replaced with
``ansible_ssh_private_key_file=workdir/id_rsa``, and you'll need to
put a (password protected) copy of your private key in your main
OnRamp directory.

The second parameter is in ``vars/main.yml``, where the **two** lines
currently reading

.. code-block::

   data_iface: ens18

need to be edited to replace ``ens18`` with the device interface for
you server. You can learn the interface using the Linux ``ip``
command:

.. code-block::

   $ ip a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host
          valid_lft forever preferred_lft forever
   2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
       link/ether 2c:f0:5d:f2:d8:21 brd ff:ff:ff:ff:ff:ff
       inet 10.76.28.113/24 metric 100 brd 10.76.28.255 scope global ens3
          valid_lft forever preferred_lft forever
       inet6 fe80::2ef0:5dff:fef2:d821/64 scope link
          valid_lft forever preferred_lft forever

In this example, the reported interface is ``ens18`` and the IP
address is ``10.76.28.113`` on subnet ``10.76.28.0/24``.  We will use
these three values as a running example throughout the Appendix, as a
placeholder for your local details.

Note that ``vars/main.yml`` and ``hosts.ini`` are the only two files
you need to modify for now, but there are additional config files that
you may want to modify as we move beyond the Quick Start deployment.
We'll identify those files throughout this section, for informational
purposes, and revisit them in later sections.


Install Ansible
~~~~~~~~~~~~~~~~~~

You need to first install Ansible before you can use it to manage your
Aether deployment. While it is possible to do this directly on your
target server, OnRamp includes a Make target to set up a Docker
container that includes a properly configured Ansible client. Start
this container by running

.. code-block::

   $ make ansible

As a result of executing this command, you will see a new prompt
that looks something like this:

.. code-block::

   root@host:/workdir#

This prompt indicates that you are running as root in the context of
the container, with ``/workdir`` as your current directory. This is
the same directory you were in when you invoked ``make``, but it is
now the root of the containerized environment. You cannot see your
actual home directory (including your ``.ssh`` directory) without
first exiting the container. To do that, type either ``exit`` or
``^D`` (Control-D).

Every time you invoke a Make command from here on, it is assumed to be
from this container (with this prompt). Because there are other
commands you will want to execute—for example, to inspect various
aspects of what you've just deployed—we recommend having two terminal
windows open on your server: one running the Ansible container (with
prompt ``root@host:/workdir#``) and one running your regular login
shell (which we designate with prompt ``$``).

Many of the tasks specified in the various Ansible playbooks result in
calls to Kubernetes, either directly (via ``kubectl``) or indirectly
(via ``helm``). This means that after executing the sequence of
Makefile targets described in the rest of this Appendix, you'll want
to run some combination of the following commands (in your regular
terminal window) to verify that the right things happened:

.. code-block::

   $ kubectl get pods --all-namespaces
   $ helm repo list
   $ helm list --namespace kube-system

The first reports the set of Kubernetes namespaces currently running;
the second shows the known set of repos you are pulling charts from;
and the third shows the version numbers of the charts currently
deployed in the ``kube-system`` namespace.

If you are not familiar with ``kubectl`` (the CLI for Kubernetes), we
recommend that you start with `Kubernetes Tutorial
<https://kubernetes.io/docs/tutorials/kubernetes-basics/>`__.  And
although not required, you may also want to install
`k9s <https://k9scli.io/>`__\ , a terminal-based UI that provides a
convenient alternative to ``kubectl`` for interacting with Kubernetes.

Note that we have not yet installed Kubernetes or Helm, so these
commands are not yet available. At this point, the only verification
step you can take is to type the following:

.. code-block::

   root@host:/workdir# make aether-pingall

The output should show that Ansible is able to securely connect to all
the nodes in your deployment, which is currently just the one that
Ansible knows as ``node1``.

Install Kubernetes
~~~~~~~~~~~~~~~~~~~

The next step is to bring up an RKE2.0 Kubernetes cluster on your
target server. Do this by typing:

.. code-block::

   root@host:/workdir# make aether-k8s-install

Once the playbook completes, executing ``kubectl`` will show the
``kube-system`` namespace running, with output looking something like
the following:

.. code-block::

   $ kubectl get pods --all-namespaces
   NAMESPACE     NAME                                                    READY   STATUS      RESTARTS   AGE 
   kube-system   cloud-controller-manager-node1                          1/1     Running     0          2m4s
   kube-system   etcd-node1                                              1/1     Running     0          104s
   kube-system   helm-install-rke2-canal-8s67r                           0/1     Completed   0          113s
   kube-system   helm-install-rke2-coredns-bk5rh                         0/1     Completed   0          113s
   kube-system   helm-install-rke2-ingress-nginx-lsjz2                   0/1     Completed   0          113s
   kube-system   helm-install-rke2-metrics-server-t8kxf                  0/1     Completed   0          113s
   kube-system   helm-install-rke2-multus-tbbhc                          0/1     Completed   0          113s
   kube-system   kube-apiserver-node1                                    1/1     Running     0          97s
   kube-system   kube-controller-manager-node1                           1/1     Running     0          2m7s
   kube-system   kube-multus-ds-96cnl                                    1/1     Running     0          95s
   kube-system   kube-proxy-node1                                        1/1     Running     0          2m1s
   kube-system   kube-scheduler-node1                                    1/1     Running     0          2m7s
   kube-system   rke2-canal-h79qq                                        2/2     Running     0          95s
   kube-system   rke2-coredns-rke2-coredns-869b5d56d4-tffjh              1/1     Running     0          95s
   kube-system   rke2-coredns-rke2-coredns-autoscaler-5b947fbb77-pj5vk   1/1     Running     0          95s
   kube-system   rke2-ingress-nginx-controller-s68rx                     1/1     Running     0          48s
   kube-system   rke2-metrics-server-6564db4569-snnv4                    1/1     Running     0          56s

Remember to run this ``kubectl`` in your regular shell, not the
Ansible container.

If you are interested in seeing the details about how Kubernetes is
customized for Aether, look at
``deps/k8s/roles/rke2/templates/master-params.yaml``.  Of particular
note, we have instructed Kubernetes to allow service for ports ranging
from ``2000`` to ``36767`` and we are using the ``multus`` and
``canal`` CNI plugins.

Install SD-Core
~~~~~~~~~~~~~~~~~~~~~~~~~

We are now ready to bring up the 5G version of the SD-Core. From
within the Ansible container type:

.. code-block::

   root@host:/workdir# make aether-5gc-install

``kubectl`` will now show the ``omec`` namespace running (in addition
to ``kube-system``), with output similar to the following:

.. code-block::

   $ kubectl get pods -n omec
   NAME                         READY   STATUS             RESTARTS      AGE
   amf-5887bbf6c5-pc9g2         1/1     Running            0             6m13s
   ausf-6dbb7655c7-42z7m        1/1     Running            0             6m13s
   kafka-0                      1/1     Running            0             6m13s
   metricfunc-b9f8c667b-r2x9g   1/1     Running            0             6m13s
   mongodb-0                    1/1     Running            0             6m13s
   mongodb-1                    1/1     Running            0             4m12s
   mongodb-arbiter-0            1/1     Running            0             6m13s
   nrf-54bf88c78c-kcm7t         1/1     Running            0             6m13s
   nssf-5b85b8978d-d29jm        1/1     Running            0             6m13s
   pcf-758d7cfb48-dwz9x         1/1     Running            0             6m13s
   sd-core-zookeeper-0          1/1     Running            0             6m13s
   simapp-6cccd6f787-jnxc7      1/1     Running            0             6m13s
   smf-7f89c6d849-wzqvx         1/1     Running            0             6m13s
   udm-768b9987b4-9qz4p         1/1     Running            0             6m13s
   udr-8566897d45-kv6zd         1/1     Running            0             6m13s
   upf-0                        5/5     Running            0             6m13s
   webui-5894ffd49d-gg2jh       1/1     Running            0             6m13s
   
You will recognize Kubernetes pods that correspond too many of the
microservices discussed is Chapter 5. For example,
``amf-5887bbf6c5-pc9g2`` implements the AMF. Note that for historical
reasons, the Aether Core is called ``omec`` instead of ``sd-core``.

If you are interested in seeing the details about how SD-Core is
configured, look at ``deps/5gc/templates/core/5g-values.yaml``.  This
is an example of a *values override* file that Helm passes to along to
Kubernetes when launching the service. Most of the default settings
will remain unchanged, with the main exception being the
``subscribers`` block of the ``omec-sub-provision`` section. This
block will eventually need to be edited to reflect the SIM cards you
actually deploy. We return to this topic in the section describing how
to bring up a physical gNB.


Run Emulated RAN Test
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can now test SD-Core with emulated traffic by typing:

.. code-block::

   root@host:/workdir# make aether-gnbsim-install
   root@host:/workdir# make aether-gnbsim-run

Note that you can re-execute the ``aether-gnbsim-run`` target multiple
times, where the results of each run are saved in a file within the
Docker container running the test. You can access that file by typing:

.. code-block::

   $ docker exec -it gnbsim-1 cat summary.log

If successful, the last lines of the output should look like the
following:

.. code-block::

   ...
   2023-04-20T20:21:36Z [INFO][GNBSIM][Profile][profile2] ExecuteProfile ended
   2023-04-20T20:21:36Z [INFO][GNBSIM][Summary] Profile Name: profile2 , Profile Type: pdusessest
   2023-04-20T20:21:36Z [INFO][GNBSIM][Summary] UEs Passed: 5 , UEs Failed: 0
   2023-04-20T20:21:36Z [INFO][GNBSIM][Summary] Profile Status: PASS

This particular test, which runs the cryptically named ``pdusessest``
profile, emulates five UEs, each of which registers with the Core,
initiates a user plane session, and then send a minimal data packet
over that session. If you are interested in the config file that
controls the test, including the option of enabling other profiles,
take a look at ``deps/gnbsim/config/gnbsim-default.yaml``. We return
to the issue of customizing gNBsim in a later section.


Clean Up
~~~~~~~~~~~~~~~~~

We recommend continuing on to the next section before wrapping up, but
when you are ready to tear down your Quick Start version Aether,
simply execute the following commands:

.. code-block::

   root@host:/workdir# make aether-gnbsim-uninstall
   root@host:/workdir# make aether-5gc-uninstall
   root@host:/workdir# make aether-k8s-uninstall

Finally, note that while we stepped through the system one component
at a time, OnRamp includes compound Make targets. For example, you
can uninstall everything covered in this section by typing:

.. code-block::

   root@host:/workdir# make aether-uninstall

Look at the ``Makefile`` to see the available set of Make targets.
