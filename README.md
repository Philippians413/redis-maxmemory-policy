### Preparation

1. Install redis
2. In first terminal window run `redis-server redis.conf` for starting redis node.
3. In second terminal window run `redis-server redis-slave.conf` for starting redis slave.
Check that master-replica connection exist:
```
$ redis-cli info replication
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=1050,lag=1
master_replid:89959e12e2190a7b0d0354290d5a4a9e8b54193e
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1050
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1050
```
4. Install [Redis random data generator](https://www.npmjs.com/package/redis-random-data-generator)

### Test maxmemory-policy volatile-lru

1. Generate data for redis
```
$ node generator.js string 100 cool \
&& node generator.js string 100 real \ 
&& node generator.js string 100 great \
&& node generator.js string 100 wonderful
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms.

127.0.0.1:6379> DBSIZE
(integer) 400

$ for i in `redis-cli KEYS 'cool*'`; do redis-cli EXPIRE $i 604800 ; done
$ for i in `redis-cli KEYS 'real*'`; do redis-cli EXPIRE $i 604800 ; done
$ for i in `redis-cli KEYS 'great*'`; do redis-cli EXPIRE $i 604800 ; done

$ for i in `redis-cli KEYS 'cool*'`; do redis-cli GET $i ; done
$ for i in `redis-cli KEYS 'cool*'`; do redis-cli GET $i ; done

$ node generator.js string 600 amazing
127.0.0.1:6379> DBSIZE
(integer) 932
```
2. Check data
```
$ redis-cli KEYS 'amazing*' | wc -l
600
$ redis-cli KEYS 'cool*' | wc -l
100
$ redis-cli KEYS 'real*' | wc -l
33
$ redis-cli KEYS 'great*' | wc -l
99
$ redis-cli KEYS 'wonderful*' | wc -l
100
```

### Test maxmemory-policy allkeys-lru

1. Generate data for redis
```
$ node generator.js string 400 cool
$ for i in `redis-cli KEYS 'cool*'`; do redis-cli GET $i ; done

$ node generator.js string 400 real
$ for i in `redis-cli KEYS 'real*'`; do redis-cli GET $i ; done

$ node generator.js string 400 great

127.0.0.1:6379> DBSIZE
(integer) 695
```
We expected to have 1200 records, but have only 695.
2. Check data
```
$ redis-cli KEYS 'great*' | wc -l
400
$ redis-cli KEYS 'cool*' | wc -l
208
$ redis-cli KEYS 'real*' | wc -l
232

127.0.0.1:6379> INFO stats
evicted_keys:505
keyspace_hits:800
keyspace_misses:0

```
As we can see 505 KEYS were evicted.

### Test maxmemory-policy volatile-lfu

1. Generate data for redis
```
$ node generator.js string 100 cool \
&& node generator.js string 100 real \ 
&& node generator.js string 100 great \
&& node generator.js string 100 wonderful
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms.

$ for i in `redis-cli KEYS '*'`; do redis-cli EXPIRE $i 604800; done
$ for i in `redis-cli KEYS 'cool*'`; do redis-cli GET $i ; done
$ for i in `redis-cli KEYS 'cool*'`; do redis-cli GET $i ; done
```
We have 400 keys that were used at least once and with 'cool' key prefix are more frequently used.
```
$ node generator.js string 700 amazing
$ for i in `redis-cli KEYS 'amazing*'`; do redis-cli EXPIRE $i 604800; done
127.0.0.1:6379> DBSIZE
(integer) 536
```
2. Check data
```
$ redis-cli KEYS 'amazing*' | wc -l
488
$ redis-cli KEYS 'great*'
(empty array)
$ redis-cli KEYS 'cool*' | wc -l
45
$ redis-cli KEYS 'real*' | wc -l
2
$ redis-cli KEYS 'wonderful*' | wc -l
1
```
As we can see keys with 'cool' key prefix are more than other, that was created before.

### Test maxmemory-policy allkeys-lfu

1. Generate data for redis
```
$ node generator.js string 100 cool \
&& node generator.js string 100 real \ 
&& node generator.js string 100 great \
&& node generator.js string 100 wonderful
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms.

127.0.0.1:6379> DBSIZE
(integer) 400

$ for i in `redis-cli KEYS 'cool*'`; do redis-cli GET $i ; done
$ for i in `redis-cli KEYS 'real*'`; do redis-cli EXPIRE $i 604800 ; done
```
We have 400 keys that were used at least once and with 'cool' key prefix are more frequently used and 'real' with EXPIRE setting.
```
$ node generator.js string 700 amazing
700 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 52ms.
127.0.0.1:6379> DBSIZE
(integer) 840
```
2. Check data
```
$ redis-cli KEYS 'amazing*' | wc -l
697
$ redis-cli KEYS 'great*' | wc -l
37
$ redis-cli KEYS 'cool*' | wc -l
37
$ redis-cli KEYS 'real*' | wc -l
33
$ redis-cli KEYS 'wonderful*' | wc -l
36
```
Wow, even distribution among all keys.

### Test maxmemory-policy volatile-random

1. Generate data for redis
```
$ node generator.js string 100 cool && \
node generator.js string 100 real && \ 
node generator.js string 100 great && \
node generator.js string 100 wonderful
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms.

127.0.0.1:6379> DBSIZE
(integer) 400

$ for i in `redis-cli KEYS 'cool*'`; do redis-cli EXPIRE $i 604800 ; done
$ for i in `redis-cli KEYS 'real*'`; do redis-cli EXPIRE $i 604800 ; done
$ for i in `redis-cli KEYS 'great*'`; do redis-cli EXPIRE $i 604800 ; done

$ node generator.js string 700 amazing
127.0.0.1:6379> DBSIZE
(integer) 816
```
2. Check data
```
$ redis-cli KEYS 'amazing*' | wc -l
700
$ redis-cli KEYS 'cool*' | wc -l
4
$ redis-cli KEYS 'real*' | wc -l
9
$ redis-cli KEYS 'great*' | wc -l
3
$ redis-cli KEYS 'wonderful*' | wc -l
100
```
As we can see KEYS without EXPIRE setting were not deleted.

### Test maxmemory-policy allkeys-random

1. Generate data for redis
```
$ node generator.js string 100 cool && \
node generator.js string 100 real && \ 
node generator.js string 100 great && \
node generator.js string 100 wonderful
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms.

127.0.0.1:6379> DBSIZE
(integer) 400

$ node generator.js string 700 amazing
127.0.0.1:6379> DBSIZE
(integer) 853
```
2. Check data
```
$ redis-cli KEYS 'amazing*' | wc -l
541
$ redis-cli KEYS 'cool*' | wc -l
73
$ redis-cli KEYS 'real*' | wc -l
82
$ redis-cli KEYS 'great*' | wc -l
80
$ redis-cli KEYS 'wonderful*' | wc -l
77
```

### Test maxmemory-policy volatile-ttl
1. Generate data for redis
```
$ node generator.js string 100 cool && \
node generator.js string 100 real && \ 
node generator.js string 100 great && \
node generator.js string 100 wonderful
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms.

127.0.0.1:6379> DBSIZE
(integer) 400

$ for i in `redis-cli KEYS 'cool*'`; do redis-cli EXPIRE $i 604800 ; done
$ for i in `redis-cli KEYS 'real*'`; do redis-cli EXPIRE $i 432000 ; done
$ for i in `redis-cli KEYS 'great*'`; do redis-cli EXPIRE $i 259200 ; done

$ node generator.js string 600 amazing
127.0.0.1:6379> DBSIZE
(integer) 811
```
2. Check data
```
$ redis-cli KEYS 'amazing*' | wc -l
600
$ redis-cli KEYS 'cool*' | wc -l
100
$ redis-cli KEYS 'real*' | wc -l
9
$ redis-cli KEYS 'great*' | wc -l
2
$ redis-cli KEYS 'wonderful*' | wc -l
100
```

### Test maxmemory-policy noeviction
1. Generate data for redis
```
$ node generator.js string 100 cool && \
node generator.js string 100 real && \ 
node generator.js string 100 great && \
node generator.js string 100 wonderful
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms. 
100 records of type string were generated and inserted [SET] into 127.0.0.1:6379 in 15ms.

127.0.0.1:6379> DBSIZE
(integer) 400

$ node generator.js string 700 amazing
127.0.0.1:6379> DBSIZE
(integer) 998
```
2. Check data
```
$ redis-cli KEYS 'amazing*' | wc -l
598
$ redis-cli KEYS 'cool*' | wc -l
100
$ redis-cli KEYS 'real*' | wc -l
100
$ redis-cli KEYS 'great*' | wc -l
100
$ redis-cli KEYS 'wonderful*' | wc -l
100
```
