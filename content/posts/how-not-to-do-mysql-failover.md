---
title: "How Not to Do Mysql Failover"
date: 2022-04-07T20:44:14+05:30
description: "fun war time story on database recovery"
---
## Prelude 

This is a story from some time ago when I accidentally messed up the MySQL failover, lost some critical data momentarily, and then managed to recover all the lost data. To begin with, some of you might ask what the heck is failover anyway? In MySQL a replicated cluster is a collective of database nodes where each of them may serve one role at a time (source & replication in MySQL's official terminology but we will stick to leader/follower) . A node can become a leader or a follower and at a time there can only be one leader (note: there can be multiple leaders with complicated topologies but let's just not talk about those in this post). Leader node receives write traffic and it replicates writes to its followers. Followers can be used for read only workload depending upon requirements. What happens if leader goes down? Well we just promote one of the followers to leader and tell the other followers about the new leader. This is called failover. At the time we did this process manually and that was the root issue of the problems in my opinion.

So first of all let me give a little bit of context before diving into the details. We were given ownership of bunch of microservices and that also meant taking care of infrastructure as and when required. We had around 100+ instances of one the microservice using few MySQL clusters for storing different things. All of the MySQL clusters had similar topology and at high lever we can imagine following kind of architecture for each database that application connected to.

![architecture](https://i.imgur.com/Re25lYz.png)

Different instances of service connected to same cluster(s) after reading topology configuration from a file. Writes went through the leader, while read went through both leader and followers. Why both? Well because there could be replication lag, if we wanted data now and delay was not acceptable for some reason then read has to go through leader. Now, the configuration file is where details about the cluster, such as replicas, leader, username, password, etc was stored and once application reads it at start up it can't be changed at runtime. So to update standard procedure was to update configuration and then restart the service instances themselves which is pretty common practice. 

Everything was working as expected when a critical hardware fault was detected on leader node and it was decided that machine has to be taken down for maintainence very soon. So we planned to do a failover when the traffic was low enough for temporary faliures to be acceptable until maintainence work was complete. Typically these things take 5-10 minutes and writes fail for the duration which is not too bad given the consequences of allowing leader to fail. 

## How we do failover anyways?

There are various automatic and manual ways to do it. We had a manual process which looked something like following

![failover](https://i.imgur.com/axkp9T7.png)

Our intial topology looks like (1). In (2) As we begin our failover, we set leader in `read_only` mode. The server permits no client updates when this flag is set. This flag was already set in all the followers to ensure nobody writes data to followers. We do this in leader to ensure data integrity while trading off write availability. As soon as we set this all the writes coming through will fail. Next in (3) we make old leader as a follower and old follower (follower 1) as new leader. Finally we tell both followers about the new leader and update the configuration file. Again Note that writes will still continue to fail because application thinks old leader is still the leader. Now to propagate changes we need to restart application instances and for those instances writes will succeed. Slowly and steadily we restart all the instances and it's usually a happy ending. Right? Well not today.

## How MySQL replication works

Let's take a very brief explaination on how MySQL replication works. First of all, leader writes events to a special log called binary log aka binlog. The binary log file stores data that followers will be reading later.

Whenever a follower connects to a leader, leader creates a new thread for the connection and then it does whatever the follower asks. Most of that is (a) feeding follower events from the binary log and (b) notifying followers about newly written events to its binary log. However, If you connect a followers that is few hours or even days behind, it will take some time to catch up.

You can see replication information by running following command on follower

```SQL

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-1.000003
          Read_Master_Log_Pos: 1462
               Relay_Log_File: slave1-relay-bin.000002
                Relay_Log_Pos: 1679
        Relay_Master_Log_File: mysql-bin-1.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
          Exec_Master_Log_Pos: 1462
              Relay_Log_Space: 1887
              Until_Condition: None
```

Interesting value is `Read_Master_Log_Pos: 1462` which tells what's the current position of follower with respect to leader. 

If we executed following command in leader we can see current position of leader itself and get an idea of replication lag.

```SQL
mysql> show master status;

+--------------------+----------+--------------+------------------+-------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+--------------------+----------+--------------+------------------+-------------------+
| mysql-bin-1.000003 |     1462 |              |                  |                   |
+--------------------+----------+--------------+------------------+-------------------+
```

(No replication lag in this example)

## The mistake

Although I've shown only two followers there were bunch of more as a result I had many SSH sessions open. As I was executing the commands late at night, I managed to turn off `read_only` in both new leader and old leader. How so? I enabled "Broadcast input to all panes in all tab" shortcut for executing same commands to all server previously and I forgot to turn it off.

![broadcast](https://i.imgur.com/k3Lhem5.png)

When I wrote following command as a final step and pressed enter, it was executed on all servers.

```SQL

mysql> SET GLOBAL read_only=OFF;

```

I didn't notice what had happened at the time and started restarting application instances to load the new configuration. Weirdly, no writes were failing now. I expected application instances who read the old configuration to at least fail for now. My face turned pale as I realized what had happened. I made both node as "leader" by allowing them to accept writes.

I had created a ["split-brain"](https://en.wikipedia.org/wiki/Split-brain_(computing)) in the cluster. Two nodes were now behaving as if they were leaders writing data independently. Acting quickly in moment of panic, I turned on read only flag in old leader. When split-brain happens you know you are in trouble because now you can't make these nodes followers/leader of each other since they basically diverged from the same binlog position. I tried anyways but `SHOW SLAVE STATUS` returned replication error. Since the old leader was going down anyway in few hours this meant we may have to discard data from the point of accident until we made it read only again. Except we can't discard data without consequences because it was very sensitive database. 

To put it visually we can see how data integrity divered from the moment accident occured

![diverge](https://i.imgur.com/1Eg04nN.png)

## Bringing back the ~~dead~~ data

This was a moment of panic but I realized that recovery might be possible. We can theoretically read the binlog from old leader and then apply that data to new leader. Just before all this happened I executed `show master status` command and I knew atleast one point in time where data was in sync. Even if I didn't I could have guessed a reasonable point back in time. The following diagram shows what data we would be replaying on the new leader for recovery. 

![recoverable](https://i.imgur.com/1ZNMqIO.png)

Although I had data intact in binlog files how the hell do I parse it? It's in binary format and I was no mood to write parser from scratch at 1 am in the morning. After a little bit of digging around I found [mysqlbinlog](https://dev.mysql.com/doc/refman/8.0/en/mysqlbinlog.html) which is a utility to interact with bin log files. 

On exploring a little bit with `--help` I found exact flag which I needed.

```SQL
  -j, --start-position=# 
   Start reading the binlog at position N. Applies to the first binlog passed on the command line.
```

This was it. You see, if you pass `-j` along with binlog file path `mysqlbinlog` parses the binlog into human readable text from that offset. It's not just human readable, it's valid SQL and you can actually execute the result on a mysql server. So that's exactly what I did. 

I ran the following command on old leader and hoped that it should restore the data....and that's what happened without a single error.

```SQL

$ mysqlbinlog -j 4512 mysql-bin-1.000004  | mysql -h <old_leader> -uroot -p

```

Bingo! Lost data was recovered back. 

![hide the pain](https://i.imgur.com/fH4gja2.jpg)

## Closing Thoughts

I think doing failover manually can be tricky because all it takes is a one rouge command to ruin everything. There are various ways to do this automatically such as  [percona-xtradb-cluster](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/manual/failover.html), [orchestrator](https://github.com/openark/orchestrator/blob/master/docs/topology-recovery.md) or even a managed database from your favourite cloud provider. So If you're reading this and you still do manual failovers please consider automating it.















