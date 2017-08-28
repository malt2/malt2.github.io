---
layout: page
title: FAQs 
order: 4 
---
<p class="message">
This section provides some insight on MALT-2 design and its usage. 
</p>


### How does MALT-2 reduce network load? 

* MALT provides sparse-reduce that performs the reduce operation with fewer workers.
Instead of synchronizing with all workers, MALT-2 provides many communication graphs
that rapidly converge and impose smaller network costs. The network graph can be passed
as an option when initializing the optimizer. MALT-2 provides various presets such as ALL,
HALTON and custom node-communication graphs. Using a sparse-graph also reduces the memory cost 
at each of the nodes since the space required to store incoming models is reduced.

* MALT-2 also has a communication batch-size (cb_size) that controls how frequently the model
averaging is performed. The default value is set to 5 mini-batches i.e. after computing
and updating the model parameters for 5 mini-batches, each replica communicates with other
models and computes an average of the nodes (as defined in the node graph).


### How does MALT-2 reduce synchronization costs?

* MALT-2 provides synchronization, asynchronization and NOTIFY_ACK synchronization methods.
To eliminate the barrier overheads for sparse reduce and to provide strong consistency, MALT-2 uses 
a NOTIFY-ACK based synchronization mechanism that gives stricter guarantees than using a coarse grained
barrier. This can also improve convergence times in some cases since it facilitates using consistent data from
dependent workers during the reduce step. In MALT-2, with NOTIFY-ACK, the parallel workers compute
and send their model parameters with notifications to other workers. They then proceed to wait to receive
notifications from all its senders as defined by their node communication graphs. The
wait operation counts the NOTIFY events and invokes the reduce when a worker has received notifications from
all its senders as described by the node communication graph. Once all notifications have been received, it can
perform a consistent reduce.

* After performing a reduce, the worker sends an ACK, indicating that the intermediate output in previous iteration
has been consumed. Only when a worker receives an ACK for a previous send, indicating that the receiver
has consumed the previously sent data, the worker may proceed to send the data for the next iteration. Unlike a
barrier based synchronization, where there is no guarantee that a receiver has consumed the intermediate outputs
from all senders, waiting on ACKs from receivers ensures that a sender never floods the receive side queue and
avoids any mixed version issues from overlapping intermediate outputs. Furthermore, fine-grained synchronization
allows efficient implementation of stochastic reduce since each sender is only blocked by dependent workers
and other workers may run asynchronously.

* NOTIFY-ACK provides clean synchronization semantics in fewer steps. Furthermore, it requires no additional
receive-side synchronization making it ideal for directmemory access style protocols such as RDMA or GPU
Direct [2]. However, NOTIFY-ACK requires ordering guarantees of the underlying implementation to guarantee
that a NOTIFY arrives after the actual data. Furthermore, in a NOTIFY-ACK based implementation, MALT-2 
ensures that the workers send their intermediate updates and then wait on their reduce inputs to avoid any
deadlock from a cyclic node communication graphs.

<center style="padding: 40px"><img width="80%" src="../ack.png" /></center>
### How does MALT-2 use Infiniband effectively?

* When a model is pushed by the sender, it appears at all its receivers (as described by the node graph),
without interrupting any of the receiver’s CPU. We expore this operation with the push API in MALT-2. Hence, model sized 
space (a receive queue) in multiples of the object size, for every sender in every machine to facilitate the push 
operation. We use per-sender receive queues to avoid invoking the receiver CPU for resolving any write-write conflicts
arising from multiple incoming model updates from different senders. Hence, our design uses extra space with the
per-sender receive queues to facilitate lockless model propagation using one-sided RDMA. Both these mechanisms,
the one sided RDMA and per-sender receive queues ensure that the scatter operation does not invoke the receive-side
CPUs making MALT-2 fully asynchronous.

