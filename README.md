# Distributed Systems Projects

Three projects built in Python with gRPC and Protocol Buffers: an online marketplace running across Google Cloud VMs, a Raft key-value store with leader leases, and K-means clustering on a MapReduce framework written from scratch.

| Folder | Project | Main files |
|---|---|---|
| [`A1/ForPython`](A1/ForPython) | Online marketplace over gRPC | `seller_service.py`, `buyer_service.py`, `*_instance*.py` |
| [`A2`](A2) | Raft with leader leases | `final_code_S1.py`, `raft_client.py`, `raft.proto`, `REPORT.pdf` |
| [`A4`](A4) | K-means on MapReduce | `master.py`, `mapper.py`, `reducer.py`, `mapReduce.proto` |

Install the shared dependencies once:

```bash
pip install grpcio grpcio-tools numpy matplotlib
```

---

## 1. Online marketplace over gRPC (`A1/ForPython`)

A marketplace where sellers list products and buyers search, buy, wishlist and rate them. Each party ran on its own Google Cloud VM and talked to the others only through gRPC.

**How it is split up**
- **Market** (`seller_service.py` on port 50053, `buyer_service.py` on port 50054) keeps the product catalogue and handles every request.
- **Sellers** (`seller_instance1.py`, `seller_instance2.py`) register, then add, update, delete and list their items (`seller.proto`).
- **Buyers** (`buyer_instance1.py`, `buyer_instance2.py`) search by name or category, buy, add to wishlist and rate items (`buyer.proto`).
- **Notifications:** every seller and buyer also runs a small gRPC server. When an item changes, the market calls back the buyers who wishlisted it. When an item is bought, it calls back the seller.

The `*_CLI.py` versions are the interactive menus used on the cloud VMs. The plain versions run a fixed demo on `localhost`.

**Running it**
```bash
cd A1/ForPython
python -m grpc_tools.protoc -I. --python_out=. --pyi_out=. --grpc_python_out=. buyer.proto seller.proto
python seller_service.py      # terminal 1
python buyer_service.py       # terminal 2
python seller_instance1.py    # terminal 3
python buyer_instance1.py     # terminal 4
```
Some addresses in `seller_service.py`, `buyer_service.py` and the `*_CLI.py` files are the public IPs of the original VMs, which no longer exist. Replace them with `localhost` or your own hosts before running.

`A1/ForJava` holds an earlier Java attempt (protos and stubs). It is not the final version.

---

## 2. Raft with leader leases (`A2`)

A replicated key-value store on 5 nodes that uses the Raft consensus algorithm. It adds **leader leases** so the leader can answer reads without a round-trip to the other nodes.

**What is implemented**
- **Leader election:** each node has a randomised election timeout of 5–10 s, then asks for votes with `RequestVote`.
- **Log replication:** the leader sends heartbeats every second through `AppendEntry`. An entry is committed once a majority has stored it.
- **Leader lease:** the leader holds a 10 s lease that it renews with every successful heartbeat round. A new leader waits for the old leader's lease to run out before it serves requests. Voters report the old lease in their `RequestVote` reply (`lease_duration`).
- **Client API:** `SET key value` and `GET key`. A node that is not the leader replies with the leader's id, so the client can redirect.
- **Persistence:** each node writes its log, metadata (term, vote, commit index) and a debug trace to `logs_node_<id>/`. A restarted node recovers its state from these files.

**Running it**
```bash
cd A2
make 0    # node 0; then make 1 … make 4, each in its own terminal or VM
python3 raft_client.py
```
The node list in `final_code_S1.py` and `raft_client.py` and the IPs in the `Makefile` point at the original Google Cloud VMs (`10.128.0.x`). Change them to `localhost` to run every node on one machine. Nodes use ports 60000+ to talk to each other and 50000+ for clients.

The `logs_node_*` folders hold logs from a real run. `REPORT.pdf` summarises the design.

---

## 3. K-means on MapReduce (`A4`)

K-means clustering written from scratch, with no ML library, on a MapReduce framework that is also written from scratch. A master, the mappers and the reducers are separate gRPC processes.

**Flow of one iteration**
1. **Split:** the master divides the indices of `points.txt` evenly across the mappers. Every mapper reads the same file and processes only its own range, so no data is sent over the network.
2. **Map:** each mapper assigns its points to the nearest centroid. It then writes one partition file per reducer (`Mappers/M<i>/partition_<r>.txt`), grouping points by centroid id.
3. **Shuffle and reduce:** each reducer fetches its partitions from every mapper through gRPC, then sorts and groups them by key. It computes the new centroid as the mean of its points and writes it to `Reducers/R<r>.txt`.
4. **Update:** the master gathers the new centroids and repeats until they stop moving or the iteration limit is hit. It then writes the final centroids to `centroids.txt`.

If a mapper or reducer fails or does not respond, the master retries its task.

**Running it**
```bash
cd A4
python3 mapper.py --port 50050    # one terminal per mapper: 50050, 50051, 50052
python3 reducer.py --port 50060   # one terminal per reducer: 50060, 50061
python3 master.py
```
The number of mappers, reducers, centroids and iterations is set at the top of `master.py` (`M = 3`, `R = 2`, `K = 2`, `N = 10`). `dump.txt` holds the console output of a full run.

---

All three were built for a Distributed Systems course. `A1` and `A2` were deployed on Google Cloud Compute Engine VMs.
