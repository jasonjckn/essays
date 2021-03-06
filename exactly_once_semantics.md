Copyright 2014 Jason Jackson. All rights reserved. 

The objective of this document to describe an implementation of “persistent aggregate” stream operator which accepts as an input stream, a key for grouping on, a monoid that implements the aggregation, and a pluggable backend storage where the results of the aggregation are persisted. The operator implemented here has exactly once semantics. Here’s an pseudo code example of using the operator:

    twitter-firehose-stream
      .flatmap( map-function =  { lambda(json) -> { for word in json.tweet_text.split_on_white_space
                                                                            yield Tuple(word, 1) },
                tuple-output-fields = [“tweet_word”, “count"] )
      .groupBy( “word” )
      .persistentAggregate(twitter-firehose, Monoid { zero() = 0, plus(a,b) = a + b }, ManhattanStore());


This is an extremely common streaming operator needed for the speed layer of summingbird jobs. Currently storm & summingbird do not compute this operation with exactly once semantics. Trident does implement this operator however it has a few synchronization points with zookeeper, kafka, and the persistent store, which means latency in any one of those 3 will add to batch latency. Additionally trident currently does one zk quorum operation per kafka partition per batch this reach zookeeper’s maximum write throughput with anywhere between 33-136 trident topologies depending on partition count and batch frequency.  Also the write load on manhattan is not uniform due to the commits to state being batched.

In this document, we describe an implementation that achieves exactly once semantics, is resilient towards swings in latency in fetching from a particular kafka broker, and kafka partition downtime. Produces a uniform load on a persistent store, where spikes in the input stream to the topology often don’t necessarily result in spikes to the back-end store, or at least soften the spike. That is resilient to wide swings in latency and throughput when writing to persistent store, or any pluggable backend store. 

The theory behind this design is based on state machines and vector clocks and monoid, I’ll review some of the theory here. Let f be a pure function that accepts input, state and produces new state: f ( input, state ) = new state. This transition function is deterministic, meaning we can always recompute any future state if we have some snapshot of older state, and can replay the input in the same order.  Replaying in the same global order in any distributed system is extremely taxing. The input stream to a streaming computation is partitioned across many kafka brokers, and you generally want to read these partitions and process them as fast as possible, which means if you replay them, you might see events in a different order next time for any particular downstream worker that is trying to deterministically compute new states from old states. So if the computation on the state is an associative and commutative operation, then as long as replay the the same input in any order, then we’ll arrive at the same state.

Let’s look at a way to describe the various orderings of input, so that in a streaming computation we get a handle of when downstream component has seen the same input as other downstream components.  We know that the kafka offsets are monotonically increasing, a particular kafka offset actually corresponds to a batch of tuples on kafka. To get even finer granularity, we can append the index of the message within it’s batch to the kafka offset (hence forth “KafkaOffsetIndex”).  We can now describe when two state machines with the same transition function has seen the same input like so: Each state machine is associated with a “clock”  (a vector of KafkaOffsetIndex). And a state machines clock c0 denotes that it’s processed all the items in each partition p0 that had an offset and index < c0[p0]. So two state machines starting from an older state and clock even if they process the input in a different order, if they have the same clock values, their states will be equal. 

Now let’s say we persist this clock value and state in a persistent store, if a computation is performing state machines transitions very quickly without persisting it with each transition, or even let’s say that a snapshot is taken place, but half way through the snapshot we have a failure; We can always recompute any future states by first doing a rollback from the last known good snapshot in the persistent storage, and recomputing from that point onward.  

It’s important to understand here that a processor can continue to transition through the state machine as quickly as it wants, including if the persistence to storage is taking a long time, or is down, the state machine can continue to handle new input coming in. However the tradeoff of persisting less frequently to a persistent storage is this leads to longer recovery times in the event the RAM state is lost. Additionally because the aggregate is a monoid, you don’t even need to keep the entire state in RAM, you seed the state machine with a clock that was the last persisted snapshot in the persistent store, but you initialize the state to zero() of the monoid, then only when it’s time to write a new snapshot to the persistent store do read in the old state of the keys that changed and do monoid plus (in-memory state, persisted state). So this computation is effectively sending just diffs to the persistent storage layer, and this can lead to very small RAM footprints. Furthermore if the persistent storage layer is down, or has high latency, you can continue to transition the state machine with new input coming in and the amount of memory consumed is only proportional to the number of new keys seen in the input. This is in effect a form of compression where if the persistent storage is down you’re compressing multiple batch database updates into one which can easily be a lot smaller in size than the size of the  than just carrying out your original batches when the persistent store comes back online. All the while you’re still keeping up with input, as oppose to idling the CPUs because one of your dependencies isn’t with real-time quality latencies like your streaming computation system. If the in-memory diff is becoming too large to hold in memory because the persistent storage layer is down, there’s all kinds of other things you can do to continue processing, or you could simply stop processing and via back propagation, the streaming computation system would pause until the persistent store comes back online to accept the diff that’s been accumulated while it was down. Furthermore, you don’t need to store every snapshot ever taken in the persistent storage, you only need store at most two snapshots (if while writing the more recent snapshot it fails part way through, you can revert to the earlier snapshot). 

Let’s look at how it’s produce a new snapshot to the persistent store. As you recall in the persistent store, we write both the state machine state ’s0', and it’s clock ‘c0', and the meaning of ‘c0’ is the state ‘s0’ is the result of having processed all the items in each kafka partition p0 that had a KafkaOffsetIndex < c0[p0]. So when streaming computation starts up it can begin reading input from each partition p0 kafka at c0[p0].kafkaOffset, since the state in the store has seen every input that has come before it. Now with a storm topology you’ll have a DAG that may be a couple layers deep with upstream bottlenecks and latencies such each downstream bolt may be processing the input that originated from each partition at differing rates, thus if we define “max_clock” (a vector of KafkaOffsetIndex) of a downstream bolt to be the KafkaOffsetIndex such that it has never seen input beyond this.  In other words max_clock is computed like so: 

    onNewInputTupleToBolt(Tuple input):
            max_clock[input.OriginatingSourceKafkaPartition] 
                = max(max_clock[input.OrigSourceKafkaPartition], input.OriginatingKafkaOffsetIndex) 

Each bolt instance will not necessarily have the same clock at the same moment in time. We don’t want to synchronize them as this will add latency to the streaming computation, so instead we have a “Snapshot Negotiator” which asks each bolt for it’s max_clock, and then merges each max_clock with merge function ‘max’ such that resulting vector of KafkaOffsetIndex describes a position in input processing that none of the bolts have yet reached, we’ll call this “desired_clock_of_next_snapshot”. It then sends desired_clock_of_next_snapshot to each of the bolts asking them to take a snapshot at this clock value. At this point the bolts which have been transitioning their state machines on a single state, will maintain two states now A and B. The current state machine state will become “A", and “B" will be initialized to monoid zero(). All input to this bolt that comes from a position that is greater than or equal to desired_clock_of_next_snapshot is merged into B, and all input that is less than desired_clock_of_next_snapshot is merged into A. Finally once min_clock (defined above) exceeds desired_clock_of_next_snapshot  in every dimension. Then we know A will never be updated again with new input and we can begin persisting A and desired_clock_of_next_snapshot to storage. The “Snapshot Negotiator” helps us have each bolt instance persist the state from the same clock, thus the persisted state is consistent with respect to the input and exactly once semantics, every pieces of input is atomically processed — even if it was split by white space and a grouping was done causing the original input tuple to travel to multiple bolt instances, the persisted state will reflect atomic processing of the kafka items.

    here's the basic algo

    bolt_prepare() {
       next_snapshot_clock = min_clock = max_clock = ManhattanStore().get_clock_of_last_snapshot().
       (A, B) = (zero(), zero())
    }

    bolt_execute(Tuple tuple) {
       low_water_mark := not defined here. 
       atomic write max_clock[tuple.partition] = max(max_clock[tuple.partition], tuple.kafkaoffsetindex)
       
       
       if(tuple.kafkaoffsetindex < low_water_mark) return; 
       if(tuple.kafkaoffsetindex > next_snapshot_clock[partition]) {
            B = plus(B, tuple)
       } else {
            A = plus(A, tuple)
       }

       if(low_water_mark > next_snapshot_clock) {
           new Thread() { take_snapshot(next_snapshot_clock, A.clone() ) }
           // clone could be optimized with persistent data structure. 
           (A, B) =  (plus(A, B), zero())
       }

    take_snapshot(clock, A) {
        ManhattanState = plus(ManhattanState, A)  // this operation could take a while, no problem. 
    }

    new Thread() {
       // snapshot negotiator thread high priority . 

      while true {
          if (snapshot_in_progress) Sleep(100ms); 
       
           while(for every element e | max_clock[e] < negotiated_clock[e] ) {
               negotiated_clock = do_negotiation(atomic read max_clock)
           }

           next_snapshot_clock = negotiated_clock
       }
    }

