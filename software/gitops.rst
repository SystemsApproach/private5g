Stage 2: GitOps Tooling
--------------------------

The Makefile targets used in Stage 1 invoke Helm to install the
applications, using application-specific *values files* found the
cloned directory (e.g., ``blueprint/latest/roc-values.yaml``) to
override the values for the corresponding Helm charts. In an
operational setting, all the information needed to deploy a set of
Kubernetes applications is checked into a Git repo, with a tool like
Fleet automatically updating the deployment whenever it detects
changes to the configuration checked into the repo.

..
  Note: There is an intermediate step that could be included. First
  use "fleet apply" locally, and then engage Fleet in the GitOps-style
  via a remote GitHub repo.

To see how this works, look at the ``resources/deploy.yaml`` file
included in the cloned directory:

.. code-block::

   apiVersion: fleet.cattle.io/v1alpha1
   kind: GitRepo
   metadata:
       name: aiab
       namespace: fleet-local
   spec:
       repo: "https://github.com/systemsapproach/aether-apps"  # Replace with your fork
       branch: main
   paths:
       - aether-2.1-alpha   # Specify one of "aether-2.0" or "aether-2.1-alpha"

This particular version uses
``https://github.com/systemsapproach/aether-apps`` as its *source repo*.
Fork that repo and then edit your local ``deploy.yaml`` to point to your
new repo. Then install Fleet on your Kubernetes cluster by typing:

.. code-block::
   
   $ make fleet-ready

Once complete, `kubectl` will show the `cattle-fleet-system` namespace
running. All that's left is to type the following command to activate Fleet:

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
automatically instantiate. This mimics how an operator would change a
deployment by checking in *Configuration-as-Code*, a practice that
proves useful when supporting live 5G workloads.

..
  Note: The set of bundles included in the *aether-apps* repo is not
  complete. Adding the missing pieces (e.g., the monitoring subsystem)
  is still work-in-progress.
