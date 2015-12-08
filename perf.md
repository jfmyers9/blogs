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
// TODO Should I expand on this? Staging/Running for CF? Draw comparisons?

While discussing the performance of the system, it is important to understand the following core components of Diego and their responsibilities.

### Cell

The cell houses the [Rep](https://github.com/cloudfoundry-incubator/rep), [Executor](https://github.com/cloudfoundry-incubator/executor) and [Garden-Linux](https://github.com/cloudfoundry-incubator/garden-linux-release).
The cell responds to container placement requests, allocates the necessary resources, and then eventually creates and runs the requested work.
It will update the actual state of LRPs and Tasks in the Database and on rolling deploys perform an evacuation process that ensures that its work load is successfully distributed across the remaining cells.

### Database

The database is the front door for interracting with Diego.
It runs the [BBS](https://github.com/cloudfoundry-incubator/bbs), a server responsible for all data access with our backing store.

### Brain

The brain operates the [Auctioneer](https://github.com/cloudfoundry-incubator/auctioneer), the internal scheduler for Diego, as well as the [Converger](https://github.com/cloudfoundry-incubator/converger), a process which over time maintains consistency between the desired and actual states of Tasks and LRPs.

## Performance Tests

In order to replace the original backend of Cloud Foundry, we needed to be confident that Diego can:
- Run a lot of stuff fast.
- Run a lot of stuff for a long time.
This resulted in the creation of two basic test suites, [fezzik](https://github.com/cloudfoundry-incubator/fezzik) and the [stress tests](https://github.com/cloudfoundry-incubator/diego-stress-tests).
Fezzik is a small test suite operating only within Diego which will spin up a large number of Tasks and LRPs proportional to the size of the environment, verify that they all run successfully, and then tear the entire work load down.
The Stress Tests are a more wholistic test suite which will use the Cloud Foundry cli to push a large number of diversified applications to saturate a Diego deployment so that we can observe the entire system under a large load for an extended period of time.

In order to measure the state of the system throughout these tests, we have set up a [Datadog dashboard](https://github.com/pivotal-cf-experimental/datadog-config-oss/blob/master/dashboard_templates/shared/diego_health_screen.json.erb) to monitor important metrics reported by the system in real time.
We have also created [Cicerone](https://github.com/cloudfoundry-incubator/cicerone) a tool that allows us to analyze the performance of the system after the fact by parsing events through logs to generate graphs that represent the lifecycles of Tasks/LRPs.
