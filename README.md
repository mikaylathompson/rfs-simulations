# RFS Simulations

This repo contains a simple simulation framework to evaluate strategies for work splitting and coordination for the Reindex-from-Snapshot tooling from [Opensearch Migrations](https://github.com/opensearch-project/opensearch-migrations). This tool probably makes no sense if you're not pretty familiar with that one.

## Strategy
THis tool simulates RFSWorkers. Each worker starts up (which takes some amount of time), attempts to grab a work item, downloads the data associated with that item (at a certain download speed), and then writes it to a (imaginary) target cluster (also at a specified write speed). It calculates a lease expiry window in the same manner as the real code -- `inital_lease_time * 2 ** num_attempts`, where `num_attempts` is the number times this particular work item has previously been picked up and released by all workers. When the lease expires, the worker stops downloading/writing the data.

In the most basic mode, when the lease expires, the worker just relinquishes its work item and shuts down. The attempt is noted on the work item, so the next time a worker grabs it, the lease time will be doubled.

A model uses a number of "universe settings" -- how fast downloads and writes are, how many shards and of what sizes are in the snapshot, etc., how many workers should be maintained, etc. to initialize the work items and the workers.

Additionally, each worker gets a "worker policy" which governs the behavior of the worker. At this point, there are only a limited number of settings available.
- `inital_lease_time_sec` - how long a worker is given for attempt 0 on a work item. Defaults to 10 minutes.
- `split_disabled` - this is the switch for "full shard" vs "sub-shard" behavior. If this is set to true (default value), the workers operate in "full shard" mode. A work item must be completed in one go, even if that means attempting it many times while the least times get longer and longer. The remaining settings only apply if this is set to false.
- `split_at_end` - There are (very high level) two subshard splitting strategies. The first is to split immediately upon downloading a large shard and then mark the current item as completed without writing any data (all of the actual "work" will be captured in the successor work items, so marking it as completed does not lose progress). The second is to download and write as much as possible, and then--upon lease expiry--split the remaining work into one or more successor work items and mark the current work item as done. This setting changes that behavior.
- `split_ratio` - Only used if `split_at_end` is true, must be in the range (0, 1]. When the lease expires, split the remaining work at this ratio. Setting this to 1 creates a "checkpoint" strategy -- each worker creates one new work item to capture all the writing it didn't finish. Setting it to 0.5 creates a binary split strategy -- each worker creates two items from every one it doesn't finish.
- `split_size_mb` - Only used if `split_at_end` is false. When a worker splits at the beginning (after downloading, before writing), it will divide the work into shards of at most this size. It functions "greedily", e.g. if the shard is 5gb and this value is 2gb, it will create successor items of sizes 2gb, 2gb, 1gb. If the work item is already equal to or smaller than this size, it will immediately write it without splitting it further.


## Limitations
### Target Cluster
By far the biggest limitation is that target cluster behavior is not included. In real life, the target cluster is often the gating element and a fickle one -- it may be able to handle 300 Mbps under ideal conditions, but if the workers attempt to write to too many shards allocated to the same node, it will throttle all of them and the total throughput may drop enormously. Right now, data write is set at the individual worker level, but not monitored at the full service level, which could easily make policies appear to scale well, when they would not in reality.

A first order approximation would be to set a maximum writing throughput for the entire cluster of workers. If the desired write level exceeds that, workers are throttled (proportionately, randomly, or greedily). E.g. if 10 workers are each trying to write at 100Mbps but the target cluster max is 300Mbps, the behaviors could be to scale each worker down to 30Mbps, to allow 3 workers to write at full speed and force the rest to pause, or to distribute the bandwidth randomly amongst the workers.

A more accurate approximation would require some way of specifying that certain initial work items are related and write to the same node. Each node could then have its own limit and workers would be throttled only if they were writing to an overloaded node. As shards are split into subshard work-items, this related-ness would need to be preserved.

Mesa supports a scheduling mode that includes both a `step` and an `advance`. Workers register their intentions during the `step` and then conflicts are reconciled during the `advance` and thsi could be used to handle gating throughput on a simple or complex basis. (source: https://github.com/projectmesa/mesa/blob/main/mesa/time.py#L179)

### Worker Randomness
The other limitation, easier to include, will be worker randomness. Some workers are on worse hardware and run X% slower across the board. Sometimes they run out of memory or disk space or otherwise crash unexpectedly. Real life download and write speeds don't match every time.

### Lease Expiry Mechanism
I haven't fully incorporated the real RFS's lease expiry mechanism. In this simulation, work items are moved between "unclaimed", "claimed", and "completed" buckets atomically. In real life, a worker claims a work item and updates the lease expiry window. Regardless of the behavior of that worker, no other worker will grab that item until its lease expiry runs out. In ideal functioning, the behavior in the simulation should match real life (a worker only drops an item if the lease has expired). If worker randomness (unexpected shutdowns) were incorporated, this would deviate. That would create situations where a work item appeared to be claimed, but wasn't actually being worked on at all.

## Usage
```
pipenv install
pipenv run jupyter notebook RFSModelAgents.ipynb
```

This should open a web browser with the Jupyter notebook.
