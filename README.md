# DataNet API Documentation
## 1 Introduction
This Python API is intended to provide users with a simple and intuitive way to access the information contained in the associated datasets. Following a user-oriented approach, the API aims to abstract users from the internal details of our datasets, thus making much easier extracting the information.

## 2 How does it work?

Beyond the complex structure of the datasets, which include many files distributed in different directories, this API provides sequentially samples (one at a time) structured in a user-friendly way. 

Two main components can be distinguished to understand how to use this API: (i) the iterator, and (ii) a sample instance. The iterator is in charge of reading and processing a sample from the dataset, while a sample is an object produced by the iterator that contains all the information of a sample in a well-structured manner.

## 3 How to use the API

We provide below a simple code snippet to initialize the iterator object of the API and obtain sequentially samples from the dataset:

````python
import datanetAPI
reader = datanetAPI.DatanetAPI(< pathToDataset >,<IntensityRange >)
it = iter(reader)
for sample in it:
  <process sample code>
````

First of all, the user needs to download and import this Python library (line 1). Then, an instance of datanetAPI can be initialized (line 2), where pathToDataset should point to the root directory of the dataset to be processed. Note that this dataset should be uncompressed in advance. IntensityRange is a Python list of integers that enables to filter only samples within a traffic intensity range. Thus, the user can specify: (i) a single value, if a specific intensity is desired, or (ii) a list with two values, that will be considered respectively as the lower and upper bounds of a range of intensities desired (e.g., IntensityRange = [800 1200] will return only the samples with traffic intensity from 800 to 1200). In a typical case, IntensityRange should be an empty list (i.e., IntensityRange = [ ]), then the iterator object will return all the samples of the dataset. Afterwards, the iterator object can be created (line 3).
Once the iterator object is created, samples can be sequentially extracted using a “for” loop (line 4). 

Alternatively, the next(it) method can be used to read only the next sample. This enables, for instance, read only “n” samples from the dataset using:

````python
for i in range(n):
    sample = next(it)
````

The next section describes in detail all the information that a sample object contains and some methods to extract it.

## 4 Structure of a Sample object

This section describes how the data is structured within a sample object.

Every sample is a container including information about: (i) a network topology, (ii) parameters used to generate the traffic in our simulation tool, (iii) a routing configuration, and (iii) some measurements (delay, jitter and loss) measured in our simulator resulting from the network configuration (i.e., topology + traffic distribution + routing). All this information is provided for every source-destination pair of the network. Note that we consider only a path connecting every source-destination (src-dst) pair, however more than a src-dst flow can traverse the same path and performance measurements can be individually obtained for each flow. Also, note that our simulator implements a method that stops the simulation when it detects that the network reaches a stationary state. Consequently, the simulation time to generate each sample is different. This is why we provide measurements such as generated or dropped packets as numbers relative to time units (e.g., packets/time unit)


More in detail, every sample instance comprises the following attributes:
* *global_packets*: Number of packets transmitted in the network per time unit (packets/time unit).
* *global_losses*: Packets lost in the network per time unit (packets/time unit).
* *global_delay*: Average per-path per-packet delay over all the packets transmitted in the network (time units).
* *maxAvgLambda*: This variable is used in our simulator to define the overall traffic intensity of the network scenario. Particularly, this is the maximum average traffic rate (bits/time unit) that a path can generate in the simulation scenario. Then, the average traffic rate of each src-dst path (lambda) is computed as 'maxAvgLambda' multiplied by a random value between 0 a 1. Note that, this traffic rate may be split into several flows sending traffic over the same src-dst path.
* *performance_matrix*: Matrix with aggregate src-dst and flow-level performance meaurements (delay, jitter and loss) measured on each src-dst pair (see more details below). 
* *traffic_matrix*: Matrix with the time and size distributions used to generate traffic for each src-dst pair (see more details below). 
* *routing_matrix*: Matrix with the paths to connect every src-dst pair (see more details below).
* *topology_object*: It uses a Graph object from the Networkx library including topology-related information at the node and link-level (see more details below).

**performance_matrix**: This is a matrix that indexes performance measurements at the level of src-dst pairs. Particularly, it considers that more than one flow can be exchanged on each src-dst pair. Hence, it provides performance measurements at two levels of granularity: (i) for all aggregate flows on each src-dst pair, and (ii) for every flow individually. Every element of this matrix (i.e., performance_matrix[src,dst]) contains a dictionary with the following keys: 
* ‘AggInfo’: dictionary with performance measurements for all aggregate flows between a specific [src,dst] pair. 
  * ‘PktsDrop’: packets dropped per time unit over the path [src,dst]. (packets/time unit)
  * ‘AvgDelay’: average per-packet delay over the packets transmitted on path [src,dst] (time units)
  * ‘p10’, ‘p20’, ‘p50’, ‘p80’, ‘p90’: percentiles (10, 20, 50, 80, and 90) of the per-packet delay distribution on path [src,dst]. (time units)
  * ‘Jitter’: Variance of the per-packet delay over the packets transmitted on path [src,dst]. This is var(packet_delay)
* ‘Flows’: List of dictionaries with performance measurements for each flow between node i and node j. Flows on each src-dst pair are indexed by a numerical ID (e.g., performance_matrix[0,1][‘Flows’][‘0’]). Note that all the flows of a src-dst pair follow the same path but may have different traffic distributions, thereby performance may vary between them. Each flow contains the following keys:
  * ‘PktsDrop’: packets dropped per time unit on the flow. (packets/time unit)
  * ‘AvgDelay’: average per-packet delay on the flow. (time units) 
  * ‘p10’, ‘p20’, ‘p50’, ‘p80’, ‘p90’: percentiles (10, 20, 50, 80, and 90) of the per-packet delay distribution on the flow. (time units)
  * ‘Jitter’: variance of the per-packet delay (i.e., jitter) on the flow. i.e., var(packet_delay)

Thus, assuming perf is the performance_matrix of a sample, we may access to the information as follows:
* *perf[src,dst]*: dictionary with the performance measurements for the communication between node src and node dst. 
* *perf[src,dst][′AggInfo′]*: dictionary with performance metrics on aggregate traffic between node src and node dst. Use *perf[src,dst][′AggInfo′][′<nameparam>′]*, where <nameparam> can be replaced by any of the keys described previously to obtain a specific performance metric (e.g., perf[0,1][′AggInfo′][′AvgDelay′]).
* *perf[src,dst][′Flows′]*: flow-level performance metrics on traffic between node src and node dst. Use *perf[src,dst][′ Flows′][′<numflow>′]* and replace <numflow> by the number ID of the desired flow to obtain its information. Extend to *perf[src,dst][′Flows′][′<numflow>′][′<nameparam>′]* and replace <nameparam> by any of the keys described previously to obtain a specific performance measurement for a specific flow (e.g., perf[0,1][′Flows′][′0′][′ PktsDrop′]).




**traffic_matrix:** This matrix indexes traffic-related information at the level of src-dst pairs. Similarly to performance_matrix, it provides traffic information at two levels of granularity: (i) for all aggregate flows on each src-dst pair, and (ii) for every flow individually. Every element of this matrix (i.e., traffic_matrix[src,dst]) contains a dictionary with the following keys:
* ‘AggInfo’: dictionary with traffic measurements for all aggregate flows between a specific [src,dst] pair. 
  * ‘AvgBw’: average bandwidth from node src to node dst (bits/time unit).
  * ‘PktsGen’: packets generated from node src to node dst per time unit (packets/time unit).
* ‘Flows’: List of dictionaries with flow-level traffic information for each flow between node src and node dst. It includes the traffic measurements for each flow and the parameters used to generate the traffic flow in our network simulator (see the different inter-packet arrival time and packet size distributions considered, and a list of parameters used in each of these distributions in Sections 4.1 and 4.2).
  * ‘TimeDis’: inter-packet arrival time distribution used to generate the traffic of the flow (see possible time distributions in Section 4.1).
  * ‘TimeDistParams’: dictionary with the parameters of the specific inter-packet arrival time distribution (see the parameters of each time distribution in Section 4.1). 
    * ‘EqLambda’: average bitrate per time unit (bits/time unit). 
    * ‘AvgPktsLambda’: average number of packets of average size generated per time unit (packets/time unit).
    * ‘ExpMaxFactor’: Factor used to define an upper bound for exponential distributions. The upper bound is defined as: ‘ExpMaxFactor’* lambda of the exponentil
  * ‘SizeDist’: packet size distribution used for the flow (see possible packet size distributions in Section 4.2).
  * ‘SizeDistParams’: dictionary with the parameters of the specific size distribution (see the parameters of each packet size distribution in Section 4.2). 
    * ‘AvgPktSize’: Average packet size (bits). 
    * ‘PktSize1’: First packet size option (bits). 
    * ‘PktSize2’: Second packet size option (bits).
  * ‘AvgBw’: average bandwidth for this flow (bits/time unit).
  * ‘PktsGen’: packets generated by this flow per time unit (packets/time unit).
  * ‘ToS’: Type of Service associated to this flow defined as an integer. 

Data within traffic_matrix is structured in a similar way as in performance_matrix. The information is indexed by every src-dst pair. However, in this case the possible parameters of the inter-packet arrival time and packet size distributions depend on the types of distribution used. For instance, in case of an exponential inter-packet time distribution, the parameters of the first flow can be accessed using the following lines of code:

traffic_matrix[src,dst][′Flows′][′0′][‘TimeDistParams’][′EqLambda’]
traffic_matrix[src,dst][′Flows′][′0′][‘TimeDistParams’][′AvgPktsLambda’]
traffic_matrix[src,dst][′Flows′][′0′][‘TimeDistParams’][′ExpMaxFactor’]

Note: If the dataset only has one flow per path, then traffic and performance measurements over the aggregate traffic on a src-dst pair are the same as flow-level measurements on the only flow in that src-dst pair. This applies to performance_matrix and traffic_matrix. For instance:
performance _matrix[0,1][′AggInfo′][′AvgDelay′]) = performance _matrix[0,1][′Flows′][′0′][′AvgDelay′])
In the case of traffic_matrix, using information at the flow-level is recommended (e.g., traffic_matrix[0,1][′Flows′][′0′]), since it includes additional information not considered at the level of aggregate traffic.

**routing_matrix**: This matrix describes the routing configuration. Particularly, it includes all the paths connecting every src-dst pair. Assuming route is a routing_matrix, “route[src,dst]” returns a list  describing the path from node src to node dst. Particularly, this list includes the IDs of the nodes that the path traverses. Note that all the flows of a src-dst pair follow the same path. 

**topology_object**: This is a Graph object form the Networkx library that provides information about the network topology. Particularly, this object encodes information on nodes and links (i.e., edges). Assuming g is a graph object of a sample instance, we can access the data as follows:
* g.nodes: Returns a list with all the nodes IDs. 
* g.edges: Returns a list of tuples describing the topology edges. Each tuple is described as (src node ID, dst node ID, link ID). The link ID is always ‘0’ as only one link for the same src-dst pair is supported at this moment.
* g[src][dst][0]: Dictionary with the information parameters of the (directed) link between node src and node dst (see more details of the link parameters in Section 4.4).

### 4.1 Parameters of time distributions
Our simulator considers the following inter-packet arrival time distributions for different flows in the network:

* TimeDist.EXPONENTIAL_T: EqLambda, AvgPktsLambda, ExpMaxFactor 

### 4.2 Parameters of packet size distributions

Our simulator considers the following packet size distributions for different flows generated in the network:

* SizeDist.BINOMIAL_S: AvgPktSize, PktSize1, PktSize2 

### 4.3 List of link parameters

All the possible node parameters are listed below:

* ‘port’: Port number of the source node that links with destination node.
* ‘bandwidth’: Link bandwidth (bits/time unit) 
* ‘weight’: Not used yet

## 5 API methods to read information from a Sample object

The API includes methods to obtain more easily some information from a sample object. This includes different matrices and dictionaries described above. Thus, the user can use the following methods (hereafter, we assume that s is a sample object obtained from the iterator):

* *s.get_global_packets()*: Returns the number of packets transmitted in the network per time unit of the sample.
* *s.get_global_losses()*: Returns the number of packets dropped in the network per time unit of the sample.
* *s.get_global_delay()*: Returns the average per-packet delay over all the packets transmitted in the network in time units of the sample.
* *s.get_maxAvgLambda()*: Returns the maximum average lambda selected to generate the traffic matrix of the sample.
* *s.get_performance_matrix()*: Returns the performance_matrix. Assuming this matrix is denoted by m, performance measurements of a specific src-dst pair can be accessed using m[src,dst]. See more details about the performance_matrix in the previous section.
* *s.get_srcdst_performance(src,dst)*: Directly returns a dictionary with the performance measurements (e.g., delay, jitter, loss) stored in performance_matrix for a particular src-dst pair. See more details about the performance_matrix in the previous section.
* *s.get_traffic_matrix()*: Returns the traffic_matrix. Assuming this matrix is denoted by m,  the information that traffic_matrix stores for a specific src-dst pair can be accessed using m[src,dst] . See more details about the traffic_matrix in the previous section.
* s.get_srcdst_traffic(src,dst): Directly returns a dictionary with information that the traffic_matrix stores for a particular src-dst pair. See more details about the traffic_matrix in the previous section.
* s.get_routing_matrix(): Returns the routing_matrix. Assuming this matrix is denoted by m, we can retrieve the path that connects the node src with node dst using m[src,dst]. See more details about the routing_matrix in the previous section.
* s.get_srcdst_routing(src,dst): Returns a list with the routing path that connects node src with node dst. 
* s.get_topology_object(): Returns a Networkx Graph object with nodes and links parameters 
* s.get_network_size(): Returns the number of nodes in the topology. 
* s.get_srcdst_link_bandwidth(src,dst): Returns the bandwidth in bits/time unit of the link between node src and node dst in case there is a link between both nodes, otherwise it returns -1.
* s.get_node_properties(node_id): Returns a dictionary with the parameters of the node identified by node_id if it exists. Otherwise it returns ‘None’. 
* s.get_link_properties(src,dst): Returns a dictionary with the parameters of the link between node src and node dst if they are connected by a link. Otherwise it returns ‘None’.

