Scale Aether
-----------------

..  Note: We assume gNBsim runs in its own server/VM from here
    forward. Current plan is to describe changes needed to bring up
    multi-node cluster here, and then discuss the details of the
    network configuration in the next section. After that is covered,
    the network-related discussion in the gNBsim and gNB sections that
    follow should then make sense.

The version of Aether used throughout the rest of this Appendix is
based on the configuration installed in this section, meaning you need
at least two nodes if you want to run gNBsim: one for gNBsim and one
for Aether. (A one-node is sufficient if you are planning to jump
ahead to physical gNBs, and additional nodes can always be added.)

**[Assuming this statement comes at the end, it would helpful to
summarize the relevant parameter settings; e.g., ``SameMachineAsCore:
false``. What else?]**

