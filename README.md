# OLDIsim

oldisim is a framework to support benchmarks that emulate Online Data-
Intensive (OLDI) workloads.

OLDI workloads are user-facing workloads that mine massive datasets across many servers
* Strict Service Level Objectives (SLO): e.g. 99%-ile tail latency is 5ms
* High fan-out with large distributed state
* Extremely challenging to perform power management

Some examples are web search and social networking.

# Run oldisim in a local cluster
## Prerequisites

The following are the required to build oldisim from this repo.

Requirements:
* SCons compiler
* C++11 compatible compiler, e.g., g++ v.4.7.3 or later
versions.
* Boost version 1.53 or higher (included).
* Cereal (included as a submodule).

Install the requirements with:
```
$ sudo apt-get install build-essential gengetopt libgoogle-perftools-dev
libunwind7-dev libevent-dev scons libboost-all-dev
```

## Build oldisim

To build oldisim, ensure that all submodules are available (`git
submodule update --init`) and run `scons` in the root directory of the project.

If you need to create static libraries, put the following in a new file named
custom.py in the project root:

```
RELEASE=1
STATICLINK=1
TCMALLOC=1
CXX='<PATH_TO_g++>'
LD='<PATH_TO_LD>'
AR='<PATH_TO_AR>'
NM='<PATH_TO_NM>'
CPPPATH=['/usr/include/', '<PATH_TO_BOOST_FILES>']
LIBPATH='/usr/lib/'
```

Note that you don’t need to build the boost library, as the dependency on lock
free queues does not require a built libboost.

To speedup compilation, scons supports parallel compilation, e.g. `scons
-j12` to compile with 12 threads in parallel. There are two build modes,
**release** and **debug**. The default build mode is **release**.
**debug** mode may be specified by passing `RELEASE=0` to `scons`, e.g. `scons
RELEASE=0`.
The output of the builds will be put into `<BUILD_MODE>/`

There are several output directories in the build, corresponding to the
different parts of oldisim.

+ *BUILD_MODE*/oldisim contains the oldisim framework libraries
+ *BUILD_MODE*/workloads contains the binaries of the workloads built

## Run oldisim: search on the cluster

This benchmark emulates the fanout and request time distribution for web search.
It models an example tree-based search topology. A user query is first processed
by a front-end server, and eventually fanned out to a set of leaf nodes.

The search benchmark consists of four modules - RootNode, LeafNode, DriverNode,
and LoadBalancer. Note that LoadBalancer is only needed when there exist more
than one root.

### Prepare the cluster

To emulate a tree topology with M roots and N leafs, your cluster needs to have
M machines to run RootNode, N machines to run LeafNode and one machine to run
DriverNode.

If M is larger than 1, one more machine is needed to enable LoadBalancer.

### Start the LeafNode

Copy the binary (release/workloads/search/LeafNode) to all the machines
allocated for LeafNode.

Run the following command:
```
$ PATH_TO_BINARY/LeafNode
```

### Start the RootNode

Copy the binary (release/workloads/search/ParentNode) to all the machines
allocated for RootNode.

Run the following command:
```
$ PATH_TO_BINARY/ParentNode --leaf=<LeafNode machine 1> ... --leaf=<LeafNode machine N>
```

### Start the LoadBalancer (optional)

Copy the binary (release/workloads/search/LoadBalancerNode) to the
machine allocated for LoadBalancerNode.

Run the following command:
```
$ PATH_TO_BINARY/LoadBalancerNode --parent=<RootNode machine 1> ... --parent=<RootNode machine M>
```

### Start the DriverNode

Copy the binary (release/workloads/search/DriverNode) to the machine
allocated for DriverNode.

Run the following command:
```
$ PATH_TO_BINARY/DriverNode --server=<RootNode machine 1> ... --server=<RootNode machine M>
```

You can run with the '--help' flag for more usage details.

# Run oldisim from PerfKitBenchmarker
Optionally you can run oldisim from the [PerfKitBenchmarker](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) using:
```
$ ./pkb.py --benchmarks=oldisim --cloud=[GCP|AZURE|AWS|...] ... --oldisim_num_leaves=[1|2|...|64] --oldisim_fanout=[1,2,...] --oldisim_latency_target=[1|2|...] --oldisim_latency_metric=[avg|50p|90p|95p|99p|99.9p]
```
## Example run on GCP
```
$ ./pkb.py --project=<GCP project ID> --benchmarks=oldisim --machine_type=f1-micro --oldisim_num_leaves=4 --oldisim_fanout=1,2,3,4 --oldisim_latency_target=40 --oldisim_latency_metric=avg
```

## Example run on AWS
```
$ ./pkb.py --cloud=AWS --benchmarks=oldisim --machine_type=t1.micro --oldisim_num_leaves=4 --oldisim_fanout=1,2,3,4 --oldisim_latency_target=40 --oldisim_latency_metric=avg
```

## Example run on Azure
```
$ ./pkb.py --cloud=Azure --machine_type=ExtraSmall --benchmarks=oldisim --oldisim_num_leaves=4 --oldisim_fanout=1,2,3,4 --oldisim_latency_target=40 --oldisim_latency_metric=avg
```
