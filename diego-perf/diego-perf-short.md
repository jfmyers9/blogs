# Diego Performance Testing

Speed. Power. Reliability... Diego.

As a container management and scheduling system, it is important that Diego not only runs containers, but also does it in a quick and timely manner.
For small work loads, it is much easier for us to guarantee that Diego adheres to these performance requirements.
However, as the work loads become larger, how do we guarantee that Diego maintains its performance characteristics?

We believe that Diego is ready to handle these production scale work loads in large distributed deployments.
As a result, we have run a series of performance tests against a 100 node Diego cluster to provide insight into how the system performs at scale.
This blog post will explore the performance tests, the measurements we made, and the future performance goals of Diego.

## What is Diego?

In understanding the performance tests, it is important to first understand what Diego is and how it operates.
Diego is a distributed container management and scheduling system designed to run arbitrary work loads.
As the backend to Cloud Foundry, it is tasked with running large amounts of processes while evenly distributing the work load across an arbitrary number of nodes.
Diego also actively monitors the health of its running work load; restarting crashed process and maintaining consistency across all nodes.

The work load on Diego can be classified into two distinct categories: Tasks and Long Running Processes (LRPs).
Tasks are one-off processes that are run to return a result to the user. i.e. Staging an applicaition on Cloud Foundry in order to produce a executable droplet.
LRPs are processes which generally do not exit unless due to failures. i.e. Running an application on Cloud Foundry such as a web server.

## Measuring Performance


