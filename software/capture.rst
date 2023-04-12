Stage 2:  Packet Capture
~~~~~~~~~~~~~~~~~~~~~~~~~

A good way to understand the inner working of Aether is to use `Ksniff
<https://github.com/eldadru/ksniff>`__ (a Kubernetes plugin) to
capture packets and display their headers as they flow into and out of
the microservices that implement Aether. Ksniff can be used with
`Wireshark <https://www.wireshark.org/>`__, but since the latter
requires a desktop display environment, we suggest starting with a
simpler setup that uses `tshark
<https://www.wireshark.org/docs/man-pages/tshark.html>`__ instead.

To install the Ksniff plugin on the server running Aether, you need to
first install the Kubernetes plugin manager: ``krew``. Cut-and-paste
the following in a shell window on your Aether server:

.. code-block::
   
    (
      set -x; cd "$(mktemp -d)" &&
      OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
      ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
      KREW="krew-${OS}_${ARCH}" &&
      curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
      tar zxvf "${KREW}.tar.gz" &&
      ./"${KREW}" install krew
    )

You also need to add ``krew`` to your search path:

.. code-block::
   
   $ export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

To verify that ``krew`` is correctly installed, run:

.. code-block::

   $ kubectl krew

You are now ready to install Ksniff:

.. code-block::

   $ kubectl krew install sniff

Finally, run Ksniff in the context of a specific Kubernetes pod by
specifying their namespace and instance names, and then redirecting
the output to ``tshark``. For example, the following captures and
displays traffic into and out of the UPF:

.. code-block::

   $ kubectl sniff -n omec upf-0 -o - | tshark -r -

As another example, you might want to sniff the ``router`` pod to see
how traffic is passed between UEs and the UPF (on the access side) and
between the UPF and the Internet (on the core side). In this case, it
can be helpful to filter the output, for example, by selecting a
specific interface on the router:

.. code-block::

    $ kubectl sniff -n default router -i access-gw -o - | tshark -r -

In this case, ``access-gw`` is the name of the router's access-side
interface.
