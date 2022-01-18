Chapter 5:  Mobile Core
============================

.. Mostly written from scratch, with the following hold-over
   content that might find a home here (including this old
   intro paragarph).

   Includes new Magma content, mostly in terms of going into much more
   detail about the cloud native implementation than we currently have.

   Includes a distributed implementation, where the User Plane
   runs at the edge (local breakout) and the Control Plane runs in the
   cloud. This is where we describe the P4-based implementation of the
   UPF.  Likely addresses the 4G / 5G / WiFi convergence story (again,
   as a side discussion).

While the aggregate functionality remains largely the same as we migrate 
from 4G to 5G, how that functionality is virtualized and factored into 
individual components changes. The 5G Mobile Core is heavily 
influenced by the cloud’s march toward a microservice-based (cloud 
native) architecture. This shift to cloud native is deeper than it might 
first appear, in part because it opens the door to customization and 
specialization. Instead of supporting just voice and broadband 
connectivity, the 5G Mobile Core can evolve to also support, for 
example, massive IoT, which has a fundamentally different latency 
requirement and usage pattern (i.e., many more devices connecting 
intermittently). This stresses—if not breaks—a one-size-fits-all 
approach to session management. 

5.1 Functional Components
-------------------------

A terminology and 3GPP-heavy intro to the functional components (for
completeness) before getting into the cloud-based implementation.

4G Mobile Core 
~~~~~~~~~~~~~~

The 4G Mobile Core, which 3GPP officially refers to as the *Evolved 
Packet Core (EPC)*, consists of five main components, the first three of 
which run in the Control Plane (CP) and the second two of which run in 
the User Plane (UP). 

-  MME (Mobility Management Entity): Tracks and manages the movement of 
   UEs throughout the RAN. This includes recording when the UE is not 
   active. 

-  HSS (Home Subscriber Server): A database that contains all 
   subscriber-related information. 

-  PCRF (Policy & Charging Rules Function): Tracks and manages policy 
   rules and records billing data on subscriber traffic. 

-  SGW (Serving Gateway): Forwards IP packets to and from the RAN. 
   Anchors the Mobile Core end of the bearer service to a (potentially 
   mobile) UE, and so is involved in handovers from one base station to 
   another. 

-  PGW (Packet Gateway): Essentially an IP router, connecting the Mobile 
   Core to the external Internet. Supports additional access-related 
   functions, including policy enforcement, traffic shaping, and 
   charging. 

Although specified as distinct components, in practice the SGW 
(RAN-facing) and PGW (Internet-facing) are often combined in a single 
device, commonly referred to as an S/PGW. The end result is illustrated 
in :numref:`Figure %s <fig-4g-core>`. 

.. _fig-4g-core:
.. figure:: figures/Slide20.png 
    :width: 700px 
    :align: center 
	    
    4G Mobile Core (Evolved Packet Core). 

Note that 3GPP is flexible in how the Mobile Core components are 
deployed to serve a geographic area. For example, a single MME/PGW pair 
might serve a metropolitan area, with SGWs deployed across ~10 edge 
sites spread throughout the city, each of which serves ~100 base 
stations. But alternative deployment configurations are allowed by the 
spec. 

5G Mobile Core 
~~~~~~~~~~~~~~

The 5G Mobile Core, which 3GPP calls the *NG-Core*, adopts a 
microservice-like architecture, where we say “microservice-like” because 
while the 3GPP specification spells out this level of disaggregation, it 
is really just prescribing a set of functional blocks and not an 
implementation. A set of functional blocks is very 
different from the collection of engineering decisions that go into 
designing a microservice-based system. That said, viewing the collection of 
components shown in :numref:`Figure %s <fig-5g-core>` 
as a set of microservices is a good working model. 

The following organizes the set of functional blocks into three groups. 
The first group runs in the Control Plane (CP) and has a counterpart in 
the EPC. 

-  AMF (Core Access and Mobility Management Function): Responsible for connection 
   and reachability management, mobility management, access 
   authentication and authorization, and location services. Manages the 
   mobility-related aspects of the EPC’s MME. 

-  SMF (Session Management Function): Manages each UE session, including 
   IP address allocation, selection of associated UP function, control 
   aspects of QoS, and control aspects of UP routing. Roughly 
   corresponds to part of the EPC’s MME and the control-related aspects 
   of the EPC’s PGW. 

-  PCF (Policy Control Function): Manages the policy rules that other CP 
   functions then enforce. Roughly corresponds to the EPC’s PCRF. 

-  UDM (Unified Data Management): Manages user identity, including the 
   generation of authentication credentials. Includes part of the 
   functionality in the EPC’s HSS. 

-  AUSF (Authentication Server Function): Essentially an authentication 
   server. Includes part of the functionality in the EPC’s HSS. 

The second group also runs in the Control Plane (CP) but does not have 
a direct counterpart in the EPC:

-  SDSF (Structured Data Storage Network Function): A “helper” service 
   used to store structured data. Could be implemented by an “SQL 
   Database” in a microservices-based system. 

-  UDSF (Unstructured Data Storage Network Function): A “helper” service 
   used to store unstructured data. Could be implemented by a “Key/Value 
   Store” in a microservices-based system. 

-  NEF (Network Exposure Function): A means to expose select 
   capabilities to third-party services, including translation between 
   internal and external representations for data. Could be implemented 
   by an “API Server” in a microservices-based system. 

-  NRF (NF Repository Function): A means to discover available services. 
   Could be implemented by a “Discovery Service” in a 
   microservices-based system. 

-  NSSF (Network Slicing Selector Function): A means to select a Network 
   Slice to serve a given UE. Network slices are essentially a way to 
   partition network resources in order to 
   differentiate service given to different users. It is a key feature 
   of 5G that we discuss in depth in a later chapter. 

The third group includes the one component that runs in the User Plane 
(UP):

-  UPF (User Plane Function): Forwards traffic between RAN and the 
   Internet, corresponding to the S/PGW combination in EPC. In addition 
   to packet forwarding, it is responsible for policy enforcement, lawful 
   intercept, traffic usage reporting, and QoS policing. 

Of these, the first and third groups are best viewed as a 
straightforward refactoring of 4G’s EPC, while the second group—despite 
the gratuitous introduction of new terminology—is 3GPP’s way of pointing 
to a cloud native solution as the desired end-state for the Mobile Core. 
Of particular note, introducing distinct storage services means that all 
the other services can be stateless, and hence, more readily scalable. 
Also note that :numref:`Figure %s <fig-5g-core>` adopts an idea that’s 
common in microservice-based systems, namely, to show a *message bus*
interconnecting all the components rather than including a full set of 
pairwise connections. This also suggests a well-understood 
implementation strategy. 

.. _fig-5g-core:
.. figure:: figures/Slide33.png 
    :width: 700px 
    :align: center 
	    
    5G Mobile Core (NG-Core). 

Stepping back from these details, and with the caveat that we are 
presuming an implementation, the main takeaway is that we can 
conceptualize the Mobile Core as a graph of services. You will 
sometimes hear this called a *Service Graph* or *Service Chain*, the 
latter being more prevalent in NFV-oriented documents. Another term,
*Service Mesh*, has taken on a rather specific meaning in cloud native 
terminology—we'll avoid overloading that term here. 3GPP is silent on 
the specific terminology since it is considered an implementation 
choice rather than part of the specification. We describe our 
implementation choices in later chapters. 


5.x Deployment Options
----------------------

.. Seems out-of-place, but maybe some of this remains (perhaps boiled
   down to a sidebar.
   
With an already deployed 4G RAN/EPC in the field and a new 5G
RAN/NG-Core deployment underway, we can’t ignore the issue of
transitioning from 4G to 5G (an issue the IP-world has been grappling
with for 20 years). 3GPP officially spells out multiple deployment
options, which can be summarized as follows.

-  Standalone 4G / Stand-Alone 5G
-  Non-Standalone (4G+5G RAN) over 4G’s EPC
-  Non-Standalone (4G+5G RAN) over 5G’s NG-Core

The second of the three options, which is generally referred to as
“NSA“, involves 5G base stations being deployed alongside the
existing 4G base stations in a given geography to provide a data-rate
and capacity boost. In NSA, control plane traffic between the user
equipment and the 4G Mobile Core utilizes (i.e., is forwarded through)
4G base stations, and the 5G base stations are used only to carry user
traffic. Eventually, it is expected that operators complete their
migration to 5G by deploying NG Core and connecting their 5G base
stations to it for Standalone (SA) operation. NSA and SA operations
are illustrated in :numref:`Figure %s <fig-nsa>`.

.. _fig-nsa:
.. figure:: figures/Slide38.png 
    :width: 600px
    :align: center
	    
    NSA and SA options for 5G deployment.

One reason we call attention to the phasing issue is that we face a
similar challenge in the chapters that follow. The closer the following
discussion gets to implementation details, the more specific we have to
be about whether we are using 4G components or 5G components. As a
general rule, we use 4G components—particularly with respect to the
Mobile Core, since that’s what's available in open source today—and trust
the reader can make the appropriate substitution without loss of
generality. Like the broader industry, the open source community is in
the process of incrementally evolving its 4G code base into its
5G-compliant counterpart.

.. _reading_migration:
.. admonition:: Further Reading

    For more insight into 4G to 5G migration strategies, see
    `Road to 5G: Introduction and Migration
    <https://www.gsma.com/futurenetworks/wp-content/uploads/2018/04/Road-to-5G-Introduction-and-Migration_FINAL.pdf>`__.
    GSMA Report, April 2018.
