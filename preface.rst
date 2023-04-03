Preface 
=======

When we wrote our introductory 5G book three years ago, our goal was
to help people with experience building Internet and cloud services to
understand the opportunity to bring best practices from those systems
to the mobile cellular network. On paper (and in the press) 5G had set
an ambitious goal of transformative changes, adopting a cloud-inspired
architecture and supporting a new set of innovative services. But the
gap between that aspirational story and the reality of 40 years of
network operators and hardware vendors protecting their incumbent
advantages made for a challenging pivot. So we started with the
basics, and set out to explain the fundamental networking concepts and
design principles behind the myriad of acronyms that dominate mobile
cellular networking.

Because 5G adopts many of the principles of cloud native systems, it
promises to bring the feature velocity of the cloud to Telco
environments. That promise is being delivered most successfully in
private 5G deployments that are less constrained by existing Telco
organizations and legacy infrastructure. What started out as sketches
on a whiteboard three years ago is now becoming a reality: Several
cloud providers are offering private 5G solutions for enterprises, and
there is a complete open source implementation of a 5G-enabled edge
cloud that the Internet community can learn from and build upon.

The architecture described in this book is not limited to private
deployments. It includes the necessary background information about
the mobile cellular network, much of which is rooted in its origin
story as a Telco voice network, but the overarching theme is to
describe the network through the lens of private deployments of 5G
connectivity as a managed cloud service. This includes adopting best
practices in horizontally scalable microservices, Software-Defined
Networking (SDN), and cloud operational practices such as DevOps.
These practices are appropriate for traditional operators, cloud
providers, and enterprises alike, but it is emerging use cases in
private deployments that will benefit first.

The book makes extensive use of open source software—specifically, the
Aether and Magma projects—to illustrate how Private 5G can be realized
in practice. The availability of open software informs our
understanding of what has historically been a proprietary and opaque
system. The result complements the low-level engineering documents
that are available online (and to which we provide links) with an
architectural roadmap for anyone trying to understand all the moving
parts, how they fit together, and how they can be operationalized.
And once you're done reading the book, we encourage you to jump into
the hands-on appendix that walks you through the step-by-step process
of deploying that software in your own local computing environment.

Acknowledgements
----------------

The software described in this book is due to the hard work of the ONF
engineering team, the Magma engineering team, and the open source
communities that work with them.

Thanks to the members of the community who contributed text or
corrections to the book, including:

- Ajay Thakur 
- Andy Bavier
- Edmar Candeia Gurjão  
- Mugahed Izzeldin
- Robert MacDavid
- Simon Leinen  

The picture of a Magma deployment in Chapter 5 was provided by Shaddi Hasan.

| Larry Peterson, Oguz Sunay, and Bruce Davie
| April 2023
