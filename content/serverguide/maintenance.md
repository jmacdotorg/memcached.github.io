+++
title = 'Maintenance'
date = 2024-09-04T13:34:32-07:00
prev = '/serverguide/configuring/'
next = '/serverguide/performance/'
weight = 2
+++

## Watching Server Health

Memcached has a lot of statistical counters. Most are tallies, but others serve as warnings for an administrator to notice. Note that there are many tools and top-like programs on the internet which serve to help you digest this information.

### Issuing Commands

Monitoring scripts should ideally issue test sets/gets/deletes to the servers occasionally, timing the response of each. If it's slow to connect you might have a connection or network problem. If it's slow to respond the server might be swapping or in ill health. You should always strictly monitor the health of a server, but an extra layer can be appreciated.

### Important Stats

These are found when issuing a simple `stats` command to a memcached server.

#### curr_connections

Lists the number of clients presently connected. Monitor that this number doesn't come too close to your max connection setting (`-c`).

#### listen_disabled_num

An obscure named statistic counting the number of times memcached has hit its connection limit. When memcached hits the max connections setting, it disables its listener and new connections will wait in a queue. When someone disconnects, memcached wakes up the listener and starts accepting again.

Each time this counter goes up, you're entering a situation where new connections _will_ lag. Make sure it stays at or close to zero.

#### accepting_conns

Related to the above listen_disabled_num, if you're already connected to memcached you can see if it's hit max connections or not by checking if this value is 1 or 0.

#### limit_maxbytes

Sometimes it's nice to ensure that how you think memcached starts and how it actually starts are in line. By checking limit_maxbytes you can verify that the `-m` argument took. Occasionally people using init scripts can be misconfigured and memcached will start with default values.

#### cmd_flush

While not exactly server health per-se, this is a good one to generically monitor. Every time someone issues a `flush_all` command, all items inside the cache are invalidated, and this counter is incremented. Sometimes debug code or misinformed people can leave scripts or callbacks running to "invalidate" the entire cache. Watch this value in production and sound the alarms if it starts moving, unless you really intended it to.

### `stats sizes`

If memcached is started with `-o track_sizes`, some extra accounting is done
internally to show a more detailed breakdown of the sizes of data stored
within memcached.

Running `stats sizes` will show how well your items align with slab classes.

WARNING: If your version of memcached is old (prior to 1.4.27) this command
can hang the cache while it scans all items. Can be very dangerous!

### Slab imbalance

A more complicated issue is an imbalance of the amount of memory assigned per slab, compared to where your actual data wants to go. Confusing speech for a simple algorithm:

 * Monitor the global 'evictions' stat. If it starts going up. (or you can skip straight to #2)
 * Monitor the `evicted` and `evicted_nonzero` stats inside `stats items`. `evicted_nonzero` means the object was evicted early and did not have an infinite expiration time. This information is per-slab.
 * Read the `total_pages` stats from `stats slabs`, and overlay those numbers with the evicted stats from above.
 * If the slabs with the most evictions line up with the slabs containing the highest number of pages, you might legitimately be out of memory.
   * Alternatively (and perhaps more important) you can correlate the per-slab hitrate with the number of pages.
 * If the slabs with the highest evictions do not line up with the pages very well, you need to restart memcached so it can re-allocate memory.

Recent versions of memcached (1.4.25 and later) have well tuned automated
features for repairing this situation without having to restart memcached. See
the release notes and
https://github.com/memcached/memcached/blob/master/doc/protocol.txt for more
information.

## Stats for Application Health

Monitoring these stats can tell you how the available memory in memcached, and how your app behaves, changes efficiency.

### Global hitrate

Hitrate is defined as: `get_hits / (get_hits + get_misses)`. The higher this value is, the more often your application is finding cache results instead of finding dead air. Watch that the value is both higher than what you expect, and watch to see if it changes with time or releases of your application.

### Hitrate per slab

While not possible to calculate the actual hitrate per-slab, since a "get miss" doesn't know how large the item _could have been_, you can monitor the number of get_hits and cmd_set on a per-slab basis via `stats slabs`. Watch that the number of get_hits is usually above cmd_set (more fetches than updates is often what you want), and watch for changes over time.

### Evictions

An item is "evicted" from the cache if it still has time to live, but ended up at the tail end of the LRU cache when it comes time to allocate a new item. While memcached tries a little to find an expired item from the tail, it doesn't guarantee it.

`stats items` shows per-slab eviction status. 'evicted' is the total number of items tossed early from that slab, 'evicted_nonzero' are the number of tossed items that did not have an unlimited expiration time, and 'evicted - evicted_nonzero' are the number of items tossed which were stored forever.

'evicted_time' notes how many seconds it's been since the last item to be evicted, was last fetched. If this number is small, it means you're evicting items which were recently used.

### Looks Can be Deceiving

You see a low hit rate, a high eviction rate, and assume that you need to buy more memory for memcached. Unfortunately this isn't necessarily true either. Consider some potential scenarios:

 * Your application sets large numbers of keys, most of which are never used again.
 * Your application tests for one or more keys on most requests that do not exist and are not recached, such a flags or values for a buggy user.

The former can cause high eviction count even though the data was not important to begin with. Audit what our application stores and see if this is a fault.

The latter can show poor hitrates, as many get_misses are accounted for by this flag. A workaround for this is to have the app "recache" a flag with a value of "not set". So the flag stays around in cache even if it's disabled, and doesn't throw off your hitrate calculations.

## OS Health (avoid swap!)

Memcached interacts hard with the network and with RAM. It's common for people to monitor swap usage, which can cause severe performance degredation to memcached.

It's also important to watch your network stack. Linux, for example, sports a number of counters for its network interfaces related to dropped packets, underruns, crc errors, etc. Most switches also have per-port counters on packet issues or link issues. Degraded performance could trace down to bad wiring, bad switchport, a dying NIC, or a network driver in need of tuning.

## Upgrading

Most minor releases of memcached focus on fixing bugs and adding instrumentation. New features are rare and tend to only happen on middle version bumps now. IE; 1.2 to 1.4 added the binary protocol and a large number of counters. 1.4.0 to 1.4.1 was mostly bugfixes and missing counters. 

Minor features may show up as well. SASL authentication support happened in the mid-1.4 series, but the change is isolated and should not break existing clients.

You should follow new releases and the release notes. It's often better to uprade early so you have the extra instrumentation when you need it, instead of wishing you had it when things go wrong.
