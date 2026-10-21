---
description: Important considerations when using Redis™ Cluster mode.
---

# Redis Cluster

Bull internals require atomic operations that span different keys. This behavior breaks Redis's rules for cluster configurations. However, it is still possible to use a cluster environment by using the proper bull prefix option as a cluster "hash tag". Hash tags are used to guarantee that certain keys are placed in the same hash slot, read more about hash tags in the [redis cluster tutorial](https://redis.io/topics/cluster-tutorial). A hash tag is defined with brackets. I.e. a key that has a substring inside brackets will use that substring to determine in which hash slot the key will be placed.

In summary, to make bull compatible with Redis cluster, use a queue prefix inside brackets. For example:

You can use two approaches in order to make the Queues compatible with Cluster. Either define a queue prefix:

```typescript
const queue = new Queue('cluster', {
  prefix: '{myprefix}',
});
```

or wrap the queue name itself:

```typescript
const queue = new Queue('{cluster}');
```

Independent queues can use different hash tags so that they can be distributed across cluster nodes, potentially increasing performance and memory usage. Queues participating in the same flow must share a hash slot, as described below.

## Flows

A flow uses atomic operations across the parent and child queues. When using Redis Cluster, all keys involved in these operations must belong to the same hash slot. Otherwise, adding the flow can fail with a `CROSSSLOT` error or a client-side error about keys belonging to different slot allocation groups. To keep different queue names in the same slot, use a common hash tag in their prefixes.

For example, use the same prefix for the flow producer and the workers that process its jobs:

```typescript
import { FlowProducer, Worker } from 'bullmq';
import { Cluster } from 'ioredis';

const connection = new Cluster([{ host: 'localhost', port: 7001 }], {
  redisOptions: { maxRetriesPerRequest: null },
});
const prefix = '{renovation}';

const flowProducer = new FlowProducer({ connection, prefix });
const stepsWorker = new Worker('steps', async job => performStep(job.data), {
  connection,
  prefix,
});
const renovateWorker = new Worker(
  'renovate',
  async job => finishRenovation(job),
  {
    connection,
    prefix,
  },
);

await flowProducer.add({
  name: 'renovate-house',
  queueName: 'renovate',
  children: [{ name: 'paint', queueName: 'steps', data: { room: 'kitchen' } }],
});
```

Here, `performStep` and `finishRenovation` are your application functions. Both queues use the `{renovation}` hash tag, so their keys belong to the same slot. The flow's jobs inherit the producer's prefix unless a node specifies its own `prefix`; any such override must keep the same hash tag. Any `Queue` instances used to access these jobs must also use the matching prefix.
