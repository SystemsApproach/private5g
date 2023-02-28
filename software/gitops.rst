Stage 2: Add GitOps Tooling
-------------------------------------

The Makefile targets used in Stage 1 invoke Helm to install the
applications, using application-specific *values files* found the
cloned directory (e.g.,
`~/systemsapproach/aether-onramp/aether-latest/roc-values.yaml`) to
override the values for the correspond Helm charts. In an operational
setting, all the information needed to deploy a set of Kubernetes
applications is checked into a Git repo, with a tool like Fleet
automatically updating the deployment whenever it detects changes to
the configuration checked into the repo.

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
       name: aiab
       namespace: fleet-local
   spec:
       repo: "https://github.com/SystemsApproach/aether-apps"  # Replace with your fork
       branch: main
   paths:
       - aether-2.1        # Specify one of "aether-2.0" or "aether-2.1"

This particular version uses
`https://github.com/SystemsApproach/aether-apps` as its *source repo*.
Fork that repo and then edit your local `deploy.yaml` to point to your
new repo. Then install Fleet on your Kubernetes cluster by typing:

.. code-block::
   
   $ make fleet-ready

Once complete, `kubectl` will show the `cattle-fleet-system` namespace
running. All that's left is to type the following command to activate Fleet:

.. code-block::
   
   $ kubectl apply -f deploy.yaml

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
automatically instantiate. This mimics how an operator would change a
deployment by checking in *Configuration-as-Code*, a practice that
proves useful when supporting live 5G workloads.

..
  Note: The set of bundles included in the *aether-apps* repo is not
  complete. Adding the missing pieces (e.g., the monitoring subsystem)
  is still work-in-progress.

To convince yourself that Fleet deploys the same artifacts as the
Makefile, note the correspondence between the Fleet-specific files in
`aether-apps` and the various configuration files in `aether-onramp`.
Using the SD-Core app in the 2.1 release of Aether as an example, you
will see that the `sd-core-5g-values.yaml` files in both `aether-apps`
and `aether-onramp` are nearly identical. Both override the default
value file supplied by the correponding Helm Chart with
deployment-specific details. The only difference is that the latter
still contains variables that the Makefile must resolve (e.g.,
`${NODE_IP}` and `${DATA_IFACE}`) before calling Helm, whereas with
the latter, you (as the operator) are responsible for specifying all
deployment-specific details.  Similarly, note that `fleet.yaml` in
`aether-apps` specifies the same information as in
`aether-onramp/configs/release-2.1` (e.g., the version of the Helm
Chart and the name of the values override file to use).  The main
difference is that `aether-apps` uses Fleet-specific syntax for the
information, whereas `aether-onramp` is ad hoc.
