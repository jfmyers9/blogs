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

- Spin up a large work load as fast as possible, and then tear it down. ([Fezzik](https://github.com/cloudfoundry-incubator/fezzik))
- Start enough applications on Diego to fill capacity and measure the performance over time. ([Stress Tests](https://github.com/cloudfoundry-incubator/diego-stress-tests))
  These stress tests also gave us an opportunity to observe a loaded Diego deployment and how it performs under various failure modes.

For our deployment under test, we wanted to replicate a larger scaler Diego deployment.
Thus, we deployed a 100 node environment on AWS, using `m3.2xlarge` with 250GB of disk for each cell node, and a `c4.4xlarge` for the database node.

## Results

From Fezzik, we were able to generate the following figures:

[ Graphs showing results from 4k Tasks + 4k LRPs Fezzik ]

From the above, we see that a 100 node Diego deployment successfully handled large, bursty workloads in a reasonable amount of time.
More specifically, Diego was able to successfully persist, schedule, run, and complete 4000 tasks and long running processes in a little under 40 seconds.
Please note, that the large bulk of time spent in these experiments was during container creation. We have since determined that this was a side effect of our choice of backing file system, and have improved container creation times drastically in future releases.

The stress tests allowed us to examine the performance of the API and bulk processing loops:

![Number of LRPs Stress Tests](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/num-lrps-stress.png "Number of LRPs Stress Tests")
![BBS Latency](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/bbs-latency.png "BBS Latency")

From the above, we can determine that the latency of requests to our single master database node increased slightly as the environment was filled, and then stabilizes at a value much less than 1 second per request.

![Bulk Loop Durations](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/bulk-loop-durations.png "Bulk Loop Durations")

We also see a very similar pattern for all of the bulk processing loops.
They all increase slightly, from their base values in an empty deployment, to reasonable values that are much less than the interval at which they are run.

[ Graphs showing LRP distribution during failure ]

During our partial failure tests (killing 5 random nodes in the deployment), we see that Diego successfully recognizes which LRP instances were missing and eventually corrects the problem by moving these applications to other cells in the deployment until all of the available space is filled.

[ Graphs showing LRP numbers during ETCD failure ]

We also see that in case of a total failure (killing the entire database cluster), Diego was able to recover and re-populate the database within a small amount of time.

[ Graphs showing total number of routes ]

During the total failure simulation, we did notice that our total number of routes dropped to a much lower number throughout the failure, rather than maintaining the original number of routes throughout the outage.
We have since determined that this was due to us incorrectly disregarding the completeness of the dataset retrieved from our database, and have corrected the problem in future releases.

Overall, we see that Diego has outperformed our expectations in almost every area we chose to observe during these performance experiments.
We also see that Diego is extremely fault tolerant, maintaining and reestablishing consistency after partial and total failure.
It is clear that there are some areas that can continued to be improved from a performance perspective, but as far as running a large scale production deployment, our tests indicate that Diego is ready for the prime time.

## The Future

While our performance experiments were conducted using a 100 node deployment, we obviously wish to support deployments of much larger scale.
However, we do not always have the resources available to run these performance experiments on large scale deployments continually.
Also, from the results above, we have identified that there are some areas of interest that did not perform up to expectations during our performance experiments.

As a result we are:
- Developing a set of [benchmark tests](https://github.com/cloudfoundry-incubator/benchmark-bbs) which will simulate the load of a much larger deployment against our database node without having to run a complete Diego deployment.
- Investigating/developing load tests for our [backing container system](https://github.com/cloudfoundry-incubator/garden) so that we can monitor and improve performance on a single cell.
- Working these performance suites into our continuous integration systems, so that we can continue to monitor the performance of the system as future changes are added to Diego.
- Respecting the completeness of the LRP dataset when updating the routing table.
- Investigating alternatives to ETCD for our backing store to further increase the stability and performance of the database node.

With the above changes, we hope to be able to validate Diego's performance at much greater scales in a short while.

If you wish to get a deeper idea into what the Diego team is working on, feel free to check out our [public project tracker](https://www.pivotaltracker.com/n/projects/1003146) or reach out to us on the [community slack channel](cloudfoundry.slack.com/messages/diego/).
