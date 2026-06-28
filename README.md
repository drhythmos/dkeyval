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
