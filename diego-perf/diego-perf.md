# The Road to 100 Cells
# Diego Performance Testing

As Diego becomes the official backend to Cloud Foundry, we need to verify that it can operate efficiently at production scale.
In an effort to prove this, we have performed a series of performance tests that has given us insight and confidence in Diego as a replacement to the DEAs.

## What Is Diego

In understanding the performance tests, it is important to first understand what Diego is, and how it operates.
Diego is a distributed container management and scheduling system designed to run arbitrary work loads.
As the backend to Cloud Foundry, it is tasked with running large amounts of processes while evenly distributing the work load across an arbitrary number of VMs.
Diego also actively monitors the health of its running work load, restarting crashed/exited process and maintaining consistency across all nodes.

The work load on Diego can be classified into two distinct categories: Tasks and Long Running Processes (LRPs).

Architecturally, Diego is made up of a few core components:

### Cell

The cell houses the [Rep](https://github.com/cloudfoundry-incubator/rep), [Executor](https://github.com/cloudfoundry-incubator/executor) and [Garden-Linux](https://github.com/cloudfoundry-incubator/garden-linux-release).
The cell responds to container placement requests, allocates the necessary resources, and then eventually creates and runs the requested work.
It will update the actual state of LRPs and Tasks in the Database and on rolling deploys perform an evacuation process that ensures that its work load is successfully distributed across the remaining cells.

### Database

The database is the front door for interracting with Diego.
It runs the [BBS](https://github.com/cloudfoundry-incubator/bbs), a server responsible for all data access with our backing store.
There is only one active BBS at any given point in time, giving Diego a single point of access to that backing store.

### Brain

The brain operates the [Auctioneer](https://github.com/cloudfoundry-incubator/auctioneer), the internal scheduler for Diego, as well as the [Converger](https://github.com/cloudfoundry-incubator/converger), a process which over time maintains consistency between the desired and actual states of Tasks and LRPs.

Diego also manages some components which are responsible for translating and interacting with the rest of the Cloud Foundry system:

### Route Emitter

The [Route Emitter](https://github.com/cloudfoundry-incubator/route-emitter) is responsible for gathering routing information LRPs running on Diego's cells and updating the routing table in CloudFoundry's [routing tier](https://github.com/cloudfoundry/gorouter).

### CC Bridge

The CC Bridge is the translation layer between the Cloud Foundry domain and Diego.
It is composed primarily of the [TPS](https://github.com/cloudfoundry-incubator/tps), [NSYNC](https://github.com/cloudfoundry-incubator/nsync), and the [Stager](https://github.com/cloudfoundry-incubator/stager), all of which help translate CF applications into tasks and LRPs at different points in an application's lifecycle.

If you wish to explore the design and architecture of Diego in more detail, please check out the documentation in the [Diego Design Notes](https://github.com/cloudfoundry-incubator/diego-design-notes#diego-design-notes).

## Performance Tests

In order to replace the original backend of Cloud Foundry, we needed to be confident that Diego can:

- Run a lot of stuff fast.
- Run a lot of stuff for a long time.

### How much is "a lot"

```
a·lot
/ā lät/
  adjective
  The amount of long running processes it takes to saturate 100 Diego cells
```

In order to properly test the performance of Diego at scale, we have to determine what size of deployment we are trying to support.
Currently Cloud Foundry deployments can range anywhere from a local bosh-lite deployment fit to run a small number of test applications to large scale production environments running tens of thousands of applications.
Therefore, in the ideal case, we would love to test Diego at a scale greater than the largest known deployment to ensure that not only can we support existing deployments, but also have confidence that they can grow over time.
However, deploying and maintaining a thousand instance deployment was unfeasible for our development team.
Therefore, we decided to chose 100 cells (10000 instances when saturated) as our target deployment size as it will provide confidence in Diego at production scale while providing insight into whether or not we can expect problems in environments that are operating at a higher load.

### The Tests

Now knowing how large a deployment we wanted to deploy, and the manner in which we wanted to test it, we were able to create two basic test suites, [fezzik](https://github.com/cloudfoundry-incubator/fezzik) and the [stress tests](https://github.com/cloudfoundry-incubator/diego-stress-tests).

Fezzik is a small test suite operating only within Diego which will spin up a large number of Tasks and LRP instances proportional to the size of the environment, verify that they all run successfully, and then tear the entire work load down.

The Stress Tests are a more wholistic test suite which will use the Cloud Foundry cli to push a large number of diversified applications to saturate a Diego deployment so that we can observe the entire system under a large load for an extended period of time.
The Stress Tests also give us an opportunity to inspect an environment under load during failure conditions (loss of cells, database failures, etc) and assert that when service returns the environment will return to a stable state.

### Measuring Performance

In order to measure the state of the system during the performance tests, we used the following tools:
- A [Datadog dashboard](https://github.com/pivotal-cf-experimental/datadog-config-oss/blob/master/dashboard_templates/shared/diego_health_screen.json.erb) to monitor important metrics reported by the system in real time.
- [Cicerone](https://github.com/cloudfoundry-incubator/cicerone), a tool which consumes application logs written by [lager](https://github.com/pivotal-golang/lager) to generate timelines and graphs of various resources throughout the system.
- `go pprof` to profile both memory and CPU performance of various components.

With these tools we are able to determine where in the system we have performance issues, and get a more granular view at how performant Diego actually is.

A more detailed explanation of the reasoning and specifics behind our performance tests can be found in our [diego development notes](https://github.com/cloudfoundry-incubator/diego-dev-notes/blob/master/proposals/measuring_performance.md).

### Areas of Interest

Before running the performance tests, we identified aspects of Diego that were in need of close attention. Specifically:

- Any bulk processing loops that exist in the system. (Route-Emitter, Nsync-Bulker, Convergence)
  As these fetch a large amount of data from the system, as the overall load on the system increases, we want to ensure that these processes do not begin to take an unreasonable amount of time.
- The single active database VM.
  As a more recent change in Diego, we needed to verify that having a single active BBS did not provide a bottleneck for data access throughout Diego.
- TLS performance throughout the system.
  
## 10 Cell Experiment

Before running our performance tests at the desired deployment size of 100 cells, we decided to run our performance suite against a smaller 10 cell deployment in order to establish confidence in our process and to identify any obvious performance degradations.
After our initial runs of fezzik, we discovered two immediate performance problems:

- The recommended size of our database VM was too small. We moved to using `m3.large` AWS instances as a result.
- The TLS configuration on our BBS server and clients were incorrect. We introduced a `ClientSessionCache` on our TLS client and Server in order to cache TLS connections and improve overall TLS performance.

Our stress tests ran with little to no problems and successfully passed all of the failure condidtions that we placed upon the system.
The results from the 10 cell experiment were promising and allowed us to iron out simple performance issues with both the test and the system itself.

## 100 Cell Experiment

With the process for running the stress tests ironed out, and success at the 10 cell deployment, we began testing our 100 cell deployment with fezzik.

### Fezzik

Initial runs of fezzik against the 100 cell deployment were plagued with some slight irregularities.
While running `go pprof` during the creation of 4000 tasks, we noticed that a large amount of CPU was being spent on TLS handshakes.
This was somewhat expected as the number of connections being made with the BBS is much greater due to the size and scale of the test as well as the deployment itself.
In order to combat this, we decided to up the size of the database VM one more time to `c4.4xlarge` giving us 16 CPU cores, 30 GB of memory, and much more computing power.
We had been expecting to make this change as we knew that having only one active BBS would force us to scale the size of the database VM with the size of the deployment.

Once we had correctly adjusted the size of the database VM to match the scale of the 100 cell deployment, we began to experience stable fezzik runs.

#### 4k Tasks

![4000 Tasks by Start Time](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/4k-tasks-by-start-time.png "4000 Tasks by Start Time")

The above graph shows the lifecycle of 4000 Tasks in Diego.
Each bar represents one distinct task, and each color represents a different phase of its lifecycle.
From this graph, we can see that Diego was able to successfully persist, run, and finish four thousand tasks in under 30 seconds on 100 cells.
The large majority of tasks were completed within under 20 seconds, while a small number of outliers took slightly longer.

One interesting pattern is the time it takes for 4000 tasks to be persisted into the BBS.
We see that the last task's creation request is not processed until about 3-4 seconds into the experiment.
After some investigation, we discovered that the `GOMAXPROCS` environment variable was not being set correctly for the BBS process.
After increasing this value to the number of cores available on the VM, we were able to process and begin persisting all 4000 tasks in less than 1.5 seconds.

Looking at the lifecycle of four thousand tasks ordered by end time gives a different perspective on the experiment.

![4000 Tasks by End Time](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/4k-tasks-by-end-time.png "4000 Tasks by End Time")

From here we can see that during this fezzik run there appears to be a bottleneck in the beginning with the light green band on the left.
After some investigation, we determined that this was due to the size of the task callback work pool, which at the time was set to only 20 workers.
We were able to eliminate this bottleneck by increasing the number of workers to a much larger number, resulted in almost no time being spent in at this point in the task lifecycle.

Another interesting pattern is that a large majority of the time is spent in the turquoise color band, which corresponds to time spent being created in [Garden](https://github.com/cloudfoundry-incubator/garden-linux-release).
Thus from Diego's perspective, we had 4000 tasks scheduled and being created in garden in under 5 seconds!

Overall the results from the four thousand task test in fezzik were extremely promising. Besides slow garden container creation performance, and two slight bottlenecks in the BBS, we were able to schedule, run, and complete a large number of tasks on 100 cells in almost no time at all.

#### 4k LRP Instances

The second major test that runs as a part of fezzik is running four thousand LRP instances.
Specifically fezzik will:
- Desire a single LRP in the system with an instance count of 4000.
- Wait until all instances report as running.
- Destroy the desired LRP from the BBS.
- Wait until all instances are removed.

Below are graphs generated by Cicerone which show the lifecycle of all 4000 LRPs in Diego.

[ INSERT LRPs ORDERED By START OR END TIME HERE ]

The above graph shows that it took around 35 seconds to persist, schedule, run, and destroy four thousand LRP instances.
One thing to note, is that the shape of the above graph is drastically different than those produced by the four thousand task experiment.
This is due to the fact that the four thousand LRP instance experiment must wait for all the instances to be running before tearing down the experiment.
Thus the we can only begin destroying instances once the last instance has been successfully created.

As with four thousand tasks, we once again see that a large majority of the time spent is during the garden container creation phase.
We also see an elevated amount of time streaming in the application bits into the container.

One concern we had with the performance of the system, was the time that it took to begin creating the actual LRP in the database, and the time that it took to stop and remove all instances from the BBS.
After investigating, we once again found that the slowness here was a result of workpools that were improperly sized for the experiment.
After increasing both the create and delete workpools for actual LRPs, we saw the experiment take much less time creating/deleteing instances.

#### Conclusions

Diego can run a lot of stuff in not a lot of time.
Besides minor complications such as the size of workpools and the size of the machine running the BBS, we did not see any major performance issues with Diego throughout the fezzik experiment.
Fezzik also drove us to expose the size of these workpools as configuration values, so that operators can properly tune them to the respective size of their environment and the load that they are expecting.

### Stress Tests
