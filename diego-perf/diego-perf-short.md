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

In order to accurately measure the results of our performance tests, we first need to define the characteristics of a performant Diego deployment.
From our experience with production scale Cloud Foundry deployments, we know that Diego must be able to:

- Run a lot of stuff fast.
- Run a lot of stuff for a long period of time.
- Run a lot of stuff successfully despite partial and total system failure.

From these performance requirements, we developed a set of metrics that will provide us insight into the state of a Cloud Foundry deployment under stress:

- Task and LRP lifecycle timelines generated from logs.
  Lifecycle timelines offer us a top level view of performance, allowing us to characterize where time was spent during the test.
- API response latency and success rate.
  The API performance of our master database node is critical to the health of a deployment.
  These metrics will allow us to validate that the API is consitently available and performant.
- Bulk operating loop durations.
  The duration of these bulk processing loops demonstrate the systems ability to maintain a consistent state.
- Total number of available routes.
  The routability of the deployment is a key metric which reflects the impact on the experience of the end user.

## Performance Tests

From our performance requirements above, we developed two testing strategies to test the performance of Diego:

- Spin up a large work load as fast as possible, and then tear it down.
- Start enough applications on Diego to fill capacity and measure the performance over time.

For our deployment under test, we wanted to replicate a larger scaler Diego deployment.
Thus, we deployed a 100 node environment on AWS, using `m3.2xlarge` for each cell node, and a `c4.4xlarge` for the database node.

## Results


