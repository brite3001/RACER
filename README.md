# RACER: A Lightweight Leaderless Consensus Algorithm for the IoT
Internet-of-Things (IoT) devices are interconnected objects embedded with sensors and software, enabling data collection and exchange. These devices encompass a wide range of applications, from household appliances to industrial systems, designed to enhance connectivity and automation. In distributed IoT networks, achieving reliable decision-making necessitates robust consensus mechanisms that allow devices to agree on a shared state of truth without reliance on central authorities. Such mechanisms are critical for ensuring system resilience under diverse operational conditions.
Recent research has identified three common limitations in existing consensus mechanisms for IoT environments: dependence on synchronised networks and clocks, reliance on centralised coordinators, and suboptimal performance. To address these challenges, this paper introduces a novel consensus mechanism called Randomised Asynchronous Consensus with Efficient Real-time Sampling (RACER). The RACER framework eliminates the need for synchronised networks and clocks by implementing the Sequenced Probabilistic Double Echo (SPDE) algorithm, which operates asynchronously without timing assumptions. Furthermore, to mitigate the reliance on centralised coordinators, RACER leverages the SPDE gossip protocol, which inherently requires no leaders, combined with a lightweight transaction ordering mechanism optimised for IoT sensor networks.
Rather than using a Blockchain for transaction ordering, we opted for an eventually consistent transaction ordering mechanism to specifically deal with high churn, asynchronous networks, and to allow devices to independently and deterministically order transactions.
To enhance the throughput of IoT networks, this paper also proposes a complementary algorithm, Peer-assisted Latency-Aware Traffic Optimisation (PLATO), designed to maximise efficiency within RACER-based systems.
The combination of RACER and PLATO is able to maintain a throughput of above 600 mb/s on a 100 node network, significantly outperforming the compared consensus mechanisms in terms of network node size and performance.

# Docker Branch
This branch contains the dockerised version of RACER. Docker allows each node to use individual threads, improving performance.

# Installation
1. Make sure you have docker installed: https://docs.docker.com/get-started/get-docker/
2. `git clone https://github.com/brite3001/phd_consensus.git`
3. `cd phd_consensus`
4. Open `make_compose_file.py` and specify the number of nodes (I'd start with 10)
5. Run `docker build -t consensus .` 
6. Run `docker compose up`

# Interpreting Logs
RACERS logs can be very noisy due to all the different nodes running in parallel. Here are some different log levels, and how to interpret them.
Log levels can be changed in the file `src/logs.py`

## ERROR Logs (PLATO Logs)
Wait about 30 seconds after running `docker compose up` and nodes will start saying `READY!!`, indicating they've completed their peer discovery. With the logger set to `ERROR` we'll only see high level debug messages for PLATO. PLATO may print messages like
- `[High RSI - /\] T: 5.265 P/FQ: 2.632 W: 6.891 O/L: 97 O/P: 100`. 
- `High RSI` indicates that the networks latency is trending upwards. 
- `T` is the self.current_latency, which is PLATOs current estimate of the networks actual latency related to data messages (seconds). 
- `P/FQ` is the self.publish_pending_frequency, which relates to the frequency at which protocol related messgaes are sent (seconds).
- `W` is the weighted_latest_latency. This values uses our latency estimate with a 60% weighting, and our peers latency estimate with a 40% weighting.
- `O/L` (our latency) is the RSI value of our nodes latency estimate. When RSI > 70 this indicates a rising trend (rising latency).
- `O/P` (our peers latency) an RSI value our node calculates based on latency estimates given by our peers.

When latency starts to decrease, PLATO will print messages like this:
- `[5.847] [Low RSI] [17] / [22] (\/) - New Target: 9.979`
- `[5.847]` is the latency of the latest data message
- `[17] / [22]` our rsi latency value is on the left, and our peers rsi latency value is on the right.
- `New Target: 9.979` This is the new reduced latency value PLATO estimates for RACER to use.

## WARNING Logs (Consensus Logs)
The warning logs track the ready and delivery phases of individual data messages. You'll see messages such as:
`Ready for: -3072406461942168690` which indicates a single node is ready for a single message
`-1491532815234411209 has been delivered!` This indicates a node has delivered a single message

## INFO Logs (Networking Logs)
This level of logging spams logs to the console, as its now printing protocol messages as they're received by nodes. You'll see messages like:
`Received EchoResponse for -6708328354717717678 from 6942964365` and
`Received ReadyResponse for 9014494830268039963 from 6942964365` which show invidiual protocol messages being passed around to nodes as part of RACER's broadcast protocol implementation.

# What values can I change to test RACER?
Remember, when making any code changes, make sure to re-run `docker build -t consensus .` before running `docker compose up`, or else changes won't be applied to the container.

Unfortunatley RACERs source code is not well documented. However, here are some values you can easily adjust:

## src/main.py
- On line 76 you can change the padding size of messages
- On line 79 you can adjust the chance the node has of sending a message
- On line 81 you can adjust the number of messages a node will batch together
- On line 87 you can adjust the sleep time between messages

## src/node.py
From line 107-113 the following variables can be adjusted
```
# Tuneable Values
target_latency: int = 2.5  # target latency for data messages. PLATO attempts to keep latency around this value.
target_publishing_frequency = 2.5  # target publishing frequency. PLATO attempts to keep publishing around this value
max_publishing_frequency = 10  # maximum publishing frequency in seconds
minimum_latency: int = 1  # minimum latency in seconds for data messages
max_gossip_timeout_time = 60  # how long before a gossip is terminated? Failed Gossip recalling and re-broadcasting not implemented.
node_selection_type = "normal"  # can choose 'poisson' 'normal' or 'random'
```
