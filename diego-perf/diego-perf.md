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

The Stress Tests are a more wholistic test suite which will use the [cf cli](https://github.com/cloudfoundry/cli) to push a large number of diversified applications to saturate a Diego deployment so that we can observe the entire system under a large load for an extended period of time.
Each type of application that is pushed puts a varying amount of load on the system by logging at different rates.
An application that crashes constantly is also pushed as part of the stress test to replicate a realistic deployment in which not every application that is pushed works.
The Stress Tests also allow us to inspect an environment under load during failure conditions (loss of cells, database failures, etc) and assert that when service returns the environment will return to a stable state.

A more detailed explanation of the reasoning and specifics behind our performance tests can be found in our [diego development notes](https://github.com/cloudfoundry-incubator/diego-dev-notes/blob/master/proposals/measuring_performance.md).

### Measuring Performance

In order to measure the state of the system during the performance tests, we used the following tools:
- A [Datadog dashboard](https://github.com/pivotal-cf-experimental/datadog-config-oss/blob/master/dashboard_templates/shared/diego_health_screen.json.erb) to monitor important metrics reported by the system in real time.
- [Cicerone](https://github.com/cloudfoundry-incubator/cicerone), a tool which consumes application logs written by [lager](https://github.com/pivotal-golang/lager) to generate timelines and graphs of various resources throughout the system.
- `go pprof` to profile both memory and CPU performance of various components.

With these tools we are able to determine where in the system we have performance issues, and get a more granular view at how performant Diego actually is.

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

![4000 LRP Instances](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/4k-lrp-instances.png "4000 LRP Instances")

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

We now need to prove that Diego can run a lot of stuff for an extended period of time.

The stress tests for a 100 cell deployment will create and run about 9.7k instances to Diego.
The initial round of CF application pushes were very successful from Diego's perspective.
We did not experience one failure from Diego's perspective.

![Stress Test LRP Instances](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-lrps.png "Stress Test LRPs")

The above graph shows the number of LRPs reported by Diego over time.
The dark blue line represents desired LRPs, the light blue line represents running LRPs, and the purple line represents crashing LRPs.
Note that the plataeu at about 8.6k instances was due to instances failing to start due to a backup at [Cloud Controller](https://github.com/cloudfoundry/cloud_controller_ng) uploading the application bits to the blobstore.
Once we had transfered every application to the `STARTED` state, we see the number of total instances reach the expected 9.7k value.

![Stress Test Cell Memory Capacity](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-cell-memory.png "Stress Cell Memory Capacity")

As expected, the available memory capacity of the 100 cells decreases over time as we saturate the deployment.
We intend to leave a small amount of space available so that during the failure tests we can successfully evacuate a good percentage of the cells successfully.

Once the environment is saturated, we checked the performance of our areas of interest highlighted above:

#### BBS Request Latency

![Stress Test BBS Latency](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-bbs-latency.png "Stress BBS Latency")

#### Cell Convergence Duration

![Stress Test Convergence Duration](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-converge-duration.png "Stress Convergence Duration")

#### Route Emitter Bulk Duration

![Stress Test Route Emitter Bulk Duration](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-route-emitter-bulk-duration.png "Stress Route Emitter Bulk Duration")

#### Rep Bulk Loop Duration

![Stress Test Rep Bulk Loop Duration](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-rep-bulk-duration.png "Stress Test Rep Bulk Loop Duration")

For every metric above we notice a similar pattern.
Each metric increased slightly over time as the environment became saturated and eventually stabilizes at an acceptable value with little variance over time.
Thus in a completely saturated environment, we did not notice any processes fail to scale with the load of the environment.

#### ETCD Performance

One area however that we did see issues with was the ETCD backing store used by the BBS.

![Stress Test ETCD Raft Term](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-etcd-raft-term-before-tuning.png "Stress Test ETCD Raft Term")

In the above graph we see that the ETCD raft term increased greatly as the store became saturated with records.
This is troubling as ETCD leader elections can cause failed writes and data access issues.

First we tried to upgrade ETCD from v2.1.2 up to v2.2.0.
This did not show any signs of improvement.

However, we found that there are two parameters that operators can tune in order to slow down this increase in raft term.
These two values are `heartbeat-interval` and `election-timeout`.
Increasing the `heartbeat-interval` and `election-timeout` values from `50 ms` and `1000 ms` to `200 ms` and `2000 ms` respectively did slow down this increase in ETCD raft term.
While it did not completely stop the ETCD leader elections, it did help alleviate the problem.

#### Conclusions

Diego can run a lot of stuff for an extended period of time.
Besides leader elections in ETCD, we saw no performance degradation in a Diego deployment under heavy load.

### Fault Tolerance

For our final experiment, we wanted to see how a Diego deployment under load performs under both partial and total failure.

#### Killing Cells

In order to simulate a partial failure, we killed 10 random cells in the deployment.
First we observed the memory capacity of the remaining cells.

![Killing Cells Memory Capacity](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-killing-cells-memory-capacity.png "Killing Cells Memory Capacity")

The destroyed cells immediately stopped reporting their capacity, and then we see the remaining capacity slowly diminish as the missing LRPs are redistributed across the remaining cells.
Killing 10 cells left the system with less capacity than was necessary to run the entire work load that was desired.
The graph of total LRPs in the system tells a similar story.

![Killing Cells LRPs](https://github.com/jfmyers9/blogs/raw/master/diego-perf/images/stress-test-killing-cells-lrps.png "Killing Cells LRPs")

As expected, we initially saw that the number of running instances drops slightly to reflect the loss of these cells.
In a short amount of time, the a large number of the previously running LRPs are restored, completely filling up all of the available space in the deployment.
This also results in the number of crashing LRPs decreasing over time as the system is no longer able to schedule them do to resource constraints.

The partial effect had no effect on the duration of any of the bulk loops in the system, and the BBS request latency stayed constant throughout the partial failure.
Thus after a partial failure, we are able to recover missing LRPs in about 2 minutes as shown by the graphs above.

#### Killing Database VMs

Killing all of the Database VMs results in a catastrophic outage in a Diego deployment.
We expect during this outage to see running instances maintain routability, however no new instances should be able to be created.
On restoration of the Database VMs, we expect to see the system return to its stable state in a short period of time.

During this experiment we noticed two concerning issues:
- A large number of the routes dropped off during the outage. We discovered that we were not respecting the "freshness" of the CF domain during the Route Emitter's bulk loop. This essentially means that the Route Emitter was taking destructive actions on an incomplete dataset.
- On restoration of the Database VM it took about 15 minutes for all LRPs to be re-persisted into ETCD. We noticed that the CPU usage on the CC Bridge VMs was extremely high during these 15 minutes. Profiling the CC Bridge processes showed us that a large majority of time was being spent generating RSA keypairs for the SSH routes.

In order to mitigate these issues in future major outages, we have changed the Route Emitter to respect the "freshness" of the CF domain, and recommended that the CC Bridge VM be run on a larger VM, similar to that of the Database VM.

With both of these changes, we no longer see routing loss during major outages, and the on recovery, the entire persistence layer is restored in less than 4 minutes.

#### Conclusions

Diego is extremely fault tolerant as a system.
In both a major and partial outage, Diego stabilizes quickly and maintains routability to running instances throughout the outage.

## Final Thoughts

Running these performance tests gave us a great deal of confidence in Diego.
It gave us an opportunity to improve performance in a good number of components.
As of [Diego Release v0.1434.0](https://github.com/cloudfoundry-incubator/diego-release/tree/v0.1434.0) we have announced Diego to be Generally Available.
It is only a matter of time before Diego becomes the official backend of Cloud Foundry.

Our next steps are to find ways to test aspects of the system at a greater scale without the burden of running deployments at such a great scale.
To solve this we have begun work on a set of [benchmark tests](https://github.com/cloudfoundry-incubator/benchmark-bbs) which will load test our Database VM at a higher level of load than we were able to create in the 100 cell deployment.
We have also created load tests for Garden-Linux to help us identify performance issues in garden container creation.
All of these new test suites are being incorporated into our continuous integration systems, so that we can ensure that Diego's performance does not degrade over time.
With the creation of these new performance suites, we should be able to have confidence in Diego at a scale of 1000+ cells.
