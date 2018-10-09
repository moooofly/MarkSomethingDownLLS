# DNS issue in AWS China

Ref: https://github.com/kubernetes/kops/issues/2858#issuecomment-315422011

So DNS ... one of our challenges. Here is some background information and what I understand.

In order to do cool stuff like HA control plane (masters) and have a functional rolling update of a cluster, you need a way for Etcd and Nodes to find the stuff it needs to find. The option that is generic i.e. works on multiple platforms, of course, is DNS. But **DNS can be a bit of a pain, and for instance, in AWS China, there is not route53 DNS**. Also what about running in GCE, when you do not want to use a public DNS domain? As far as I know, a user cannot have a private domain hosted by GCE.

So enter `gossip`. As mentioned above `gossip` is a communication pattern which allows a distributed system to maintain eventual consistency. `Gossip` is actually a very good pattern for eventual consistency. Many systems like **Weave** and **Cassandra** use `gossip`.

Quite simply instead of using DNS, **a kops clusters can perform lookups using `gossip`**. Here is an example. During the creation of a kops cluster or a rolling update of a cluster, Etcd nodes need to discover each other. Instead of using Route53 for DNS, the hostnames are now updated via protokube container running `gossip`.

Trade offs. At this point using `gossip` is new, and is not hardened by users breaking it. So that is exactly what we need the user base to do, is use it, break it for us, so we can make kops great! Also without DNS, you do not get stuff like a DNS name for the API endpoint.