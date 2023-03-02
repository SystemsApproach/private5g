Stage 2: Add GitOps Tooling
-------------------------------------

The Makefile targets used in Stage 1 invoke Helm to install the
applications, using application-specific *values files* found the
cloned directory (e.g.,
`~/systemsapproach/aether-onramp/aether-latest/roc-values.yaml`) to
override the default values for the corresponding Helm Charts. In an
operational setting, all such information is checked into a Git repo,
with a tool like Fleet automatically updating the deployment whenever
it detects changes to the configuration checked into the repo.

..
  Note: There is an intermediate step that could be included. First
  use "fleet apply" locally, and then engage Fleet in the GitOps-style
  via a remote GitHub repo.

To see how this works, look at the `resources/deploy.yaml` file
included in the cloned directory:

.. code-block::

   apiVersion: fleet.cattle.io/v1alpha1
   kind: GitRepo
   metadata:
       name: aether
       namespace: fleet-local
   spec:
       repo: "https://github.com/SystemsApproach/aether-apps"  # Replace with your fork
       branch: main
   paths:
       - aether-2.1        # Specify one of "aether-2.0" or "aether-2.1"

This particular version uses
`https://github.com/SystemsApproach/aether-apps` as its *source repo*.
Fork that repo and then edit your local `deploy.yaml` to point to this
new repo. Then install Fleet on your Kubernetes cluster by typing:

.. code-block::
   
   $ make fleet-ready

Once complete, `kubectl` will show the `cattle-fleet-system` namespace
running. All that's left is to activate Fleet on your cluster, but
before doing that, you first need to edit the Fleet specifications in
your forked `aether-apps` repo to reflect the details of your
particular deployment. This is a manual process, but one worth
understanding because it is how will eventually manage an
operational deployment.

A good starting point is to compare the Fleet-specific files in
`aether-apps` with various configuration files in `aether-onramp`.
Using the SD-Core app in the 2.1 release of Aether as an example, you
will see that the respective `sd-core-5g-values.yaml` files in
`aether-apps` and `aether-onramp` are nearly identical. One difference
is that the latter contains variables that the Makefile has to resolve
(e.g., `${NODE_IP}`, `${DATA_IFACE}`, and `${ENABLE_GNBSIM}`) before
calling Helm, whereas the former has default values already filled in.
These default values are sufficient for getting started (e.g., it
enables the GNBSIM emulator), but they will need to change as your
deployment environment becomes more complex, or you want to
reconfigure the respective Kubernetes applications.

Note that `aether-apps` also has a set of `fleet.yaml` files, each of
which contains information that can also be found in
`aether-onramp/configs/release-2.1` (e.g., the Helm Chart version
number and the name of the values override file).  The main difference
is that `aether-apps` uses Fleet-specific syntax for this information,
whereas `aether-onramp` is ad hoc. This information does not need to
be edited until you are ready to substitute a different values file or
chart version.

You are now ready to activate Fleet, which can be done by typing:

.. code-block::
   
   $ kubectl apply -f resources/deploy.yaml

The following command will let you track Fleet as it makes progress
installing applications (which Fleet refers to as *bundles*):

.. code-block::
   
   $ kubectl -n fleet-local get bundles

Once complete, you can rerun the same emulated 5G test against Aether:

.. code-block::

   $ make 5g-test

Once you configure your cluster to use Fleet to deploy the Kubernetes
applications, the "clean" targets in the Makefile will no longer work
correctly: *Fleet will persist in reinstalling any namespaces that have
been deleted.* You have to first uninstall Fleet by typing:

.. code-block::

   $ make fleet-clean
   
before executing the other "clean" targets. Alternatively, leave Fleet
running and instead modify your forked copy of the `aether-apps` repo
to no longer include applications you do not want Fleet to
automatically instantiate. This mimics how an operator would update a
deployment or support multiple distinct deployments (e.g.,
"production" and "development"), but it is cumbersome as long as you
are in exploratory mode, so you may want to disable Fleet until you
are ready for that level of operational overhead.
