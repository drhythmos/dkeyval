# keyval

keyval is a reference example use of the Hashicorp Raft implementation. [Raft](https://raft.github.io/) is a _distributed consensus protocol_, meaning its purpose is to ensure that a set of nodes -- a cluster -- agree on the state of some arbitrary state machine, even when nodes are vulnerable to failure and network partitions. Distributed consensus is a fundamental concept when it comes to building fault-tolerant systems.

## Reading and writing keys

The reference implementation is a very simple in-memory key-value store. You can set a key by sending a request to the HTTP bind address (which defaults to `localhost:11000`):

```bash
curl -XPOST localhost:11000/key -d '{"foo": "bar"}'
```

You can read the value for a key like so:

```bash
curl -XGET localhost:11000/key/foo
```

## Running keyval

Starting and running a keyval cluster is easy. Download and build keyval like so:

```bash
mkdir project # or any directory you like
cd project
export GOPATH=$PWD
mkdir -p src/github.com/forkbikash
cd src/github.com/forkbikash/
git clone https://github.com/forkbikash/keyval.git
cd keyval
go install
```

Run your first keyval node like so:

```bash
$GOPATH/bin/keyval -id node0 ~/node0
```

You can now set a key and read its value back:

```bash
curl -XPOST localhost:11000/key -d '{"user1": "batman"}'
curl -XGET localhost:11000/key/user1
```

### Bring up a cluster

_A walkthrough of setting up a more realistic cluster is [here](https://github.com/forkbikash/keyval/blob/main/CLUSTERING.md)._

Let's bring up 2 more nodes, so we have a 3-node cluster. That way we can tolerate the failure of 1 node:

```bash
$GOPATH/bin/keyval -id node1 -haddr localhost:11001 -raddr localhost:12001 -join :11000 ~/node1
$GOPATH/bin/keyval -id node2 -haddr localhost:11002 -raddr localhost:12002 -join :11000 ~/node2
```

_This example shows each keyval node running on the same host, so each node must listen on different ports. This would not be necessary if each node ran on a different host._

This tells each new node to join the existing node. Once joined, each node now knows about the key:

```bash
curl -XGET localhost:11000/key/user1
curl -XGET localhost:11001/key/user1
curl -XGET localhost:11002/key/user1
```

Furthermore you can add a second key:

```bash
curl -XPOST localhost:11000/key -d '{"user2": "robin"}'
```

Confirm that the new key has been set like so:

```bash
curl -XGET localhost:11000/key/user2
curl -XGET localhost:11001/key/user2
curl -XGET localhost:11002/key/user2
```

#### Stale reads

Because any node will answer a GET request, and nodes may "fall behind" updates, stale reads are possible. Again, keyval is a simple program, for the purpose of demonstrating a distributed key-value store.

### Tolerating failure

Kill the leader process and watch one of the other nodes be elected leader. The keys are still available for query on the other nodes, and you can set keys on the new leader. Furthermore, when the first node is restarted, it will rejoin the cluster and learn about any updates that occurred while it was down.

A 3-node cluster can tolerate the failure of a single node, but a 5-node cluster can tolerate the failure of two nodes. But 5-node clusters require that the leader contact a larger number of nodes before any change e.g. setting a key's value, can be considered committed.

### Leader-forwarding

Automatically forwarding requests to set keys to the current leader is not implemented. The client must always send requests to change a key to the leader or an error will be returned.


# KeyVal (Hashicorp Raft) - Setup & Demo Guide

This guide explains how to run a 3-node KeyVal cluster using the Hashicorp Raft implementation on **Windows + WSL (Ubuntu)**.

---

## Prerequisites

- Windows + WSL (Ubuntu)
- Go installed
- Project located at:

```text
/mnt/c/Users/rakes/keyval
```

---

# Step 1: Open WSL

```bash
wsl
```

---

# Step 2: Navigate to the Project

```bash
cd /mnt/c/Users/rakes/keyval
```

Verify:

```bash
pwd
ls
```

---

# Step 3: Stop Existing Processes

```bash
pkill keyval
```

---

# Step 4: Delete Old Cluster Data (Fresh Start)

```bash
rm -rf node0 node1 node2
```

Verify:

```bash
ls
```

You should NOT see:

```text
node0
node1
node2
```

---

# Step 5: Build the Project (Optional)

Run only if you modified the source code.

```bash
go build -o keyval .
```

---

# Step 6: Start Node 0 (Leader)

Open **Terminal 1**.

```bash
wsl

cd /mnt/c/Users/rakes/keyval

./keyval -id node0 ./node0
```

Expected logs:

```text
keyval started successfully, listening on http://localhost:11000

...

entering leader state
```

Leave this terminal running.

---

# Step 7: Start Node 1

Open **Terminal 2**.

```bash
wsl

cd /mnt/c/Users/rakes/keyval

./keyval \
-id node1 \
-haddr localhost:11001 \
-raddr localhost:12001 \
-join :11000 \
./node1
```

Leave this terminal running.

---

# Step 8: Start Node 2

Open **Terminal 3**.

```bash
wsl

cd /mnt/c/Users/rakes/keyval

./keyval \
-id node2 \
-haddr localhost:11002 \
-raddr localhost:12002 \
-join :11000 \
./node2
```

Leave this terminal running.

---

# Step 9: Store a Key

Open **Terminal 4**.

```bash
wsl

cd /mnt/c/Users/rakes/keyval
```

Store a key:

```bash
curl -i -X POST http://localhost:11000/key \
-d '{"user1":"batman"}'
```

Expected:

```text
HTTP/1.1 200 OK
Content-Length: 0
```

---

# Step 10: Verify Replication

Read from the leader:

```bash
curl http://localhost:11000/key/user1
```

Read from Node 1:

```bash
curl http://localhost:11001/key/user1
```

Read from Node 2:

```bash
curl http://localhost:11002/key/user1
```

Expected output from all nodes:

```json
{"user1":"batman"}
```

---

# Step 11: Add Another Key

```bash
curl -i -X POST http://localhost:11000/key \
-d '{"user2":"robin"}'
```

Verify:

```bash
curl http://localhost:11000/key/user2

curl http://localhost:11001/key/user2

curl http://localhost:11002/key/user2
```

Expected:

```json
{"user2":"robin"}
```

---

# Step 12: Test Leader Failure

Go to the terminal running **Node 0**.

Press:

```text
Ctrl + C
```

Wait about **5–10 seconds**.

One of the remaining nodes will print logs similar to:

```text
entering candidate state
election won
entering leader state
```

That node is now the **new leader**.

---

# Step 13: Write to the New Leader

If **Node 1** becomes the leader:

```bash
curl -i -X POST http://localhost:11001/key \
-d '{"user3":"joker"}'
```

If **Node 2** becomes the leader:

```bash
curl -i -X POST http://localhost:11002/key \
-d '{"user3":"joker"}'
```

Verify:

```bash
curl http://localhost:11001/key/user3

curl http://localhost:11002/key/user3
```

Expected:

```json
{"user3":"joker"}
```

---

# Step 14: Restart Node 0

Open a new terminal.

```bash
wsl

cd /mnt/c/Users/rakes/keyval

./keyval -id node0 ./node0
```

Verify synchronization:

```bash
curl http://localhost:11000/key/user3
```

Expected:

```json
{"user3":"joker"}
```

---

# Important Notes

- Always send **POST** requests to the **current leader**.
- Followers do **not** forward write requests.
- Writing to a follower returns:

```text
HTTP/1.1 500 Internal Server Error
```

- **GET** requests can be served by any node.
- A **3-node cluster** can tolerate the failure of **one node**.
- Restarted nodes automatically rejoin the cluster and synchronize with the leader.

---

# Reset the Cluster

Stop all running nodes:

```bash
pkill keyval
```

Delete cluster data:

```bash
rm -rf node0 node1 node2
```

(Optional) Rebuild:

```bash
go build -o keyval .
```

Start again from **Step 6**.
