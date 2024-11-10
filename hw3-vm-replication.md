# Homework 3---VM Replication

**Author: Cayden Lund (u1182408)**

This homework assignment asks us to discuss Remus, a piece of software that replicates a virtual machine asynchronously.
I will discuss the way that Remus works, go over scenarios that break consistency of execution, and explain the crash consistency of the disk.


## How Remus Works

The Remus software runs on two servers.
One of the servers hosts the "active" virtual machine, which is the one running the application and handling requests.
The other server hosts the "passive" virtual machine, a replica of the active virtual machine, which is not running; rather, it is a complete, in-memory image of the primary machine that can be activated instantly at any moment.

Remus accomplishes this by making frequent snapshots of the active virtual machine (every 25 ms), which captures all critical system states that need to be replicated.
To ensure that the data integrity is upheld, it pauses I/O (including network packets and disk reads/writes) while the snapshot is being taken, but these snapshots are taken so quickly that clients and the virtual system hardly notice.
Then, the snapshot is conveyed asynchronously in the background.
The asynchronous snapshotting has the enormous benefit that the active system doesn't need to wait for the backup system to catch up; it simply continues running like normal.
This is called **speculative execution**, because the execution state might be lost if the system crashes before the next checkpoint is taken.

The other key concept to Remus is **buffered I/O**.
Whenever network arrive or are transmitted, and whenever the disk is read or written, all I/O passes through the buffer.
The process of committing I/O happens in a few stages; first, once data is ready to be transmitted, the data in the buffer is marked as in-progress and synchronized between the active and passive system.
Once synchronized, the data is transmitted, written to the disk, or sent to the application as appropriate.
Once the changes have been made, the data in the buffer is marked as complete, and once again synchronized between the active and passive systems.

If the active system fails, then the passive system is immediately activated.
The latency of activating and switching to the passive system is impressively small (usually 1 second or less), and the application state from the last checkpoint is perfectly preserved.
This means that for most applications, clients won't even disconnect or lose any data.
If there were any I/O operations in progress, then they will still be in the buffer and not marked as complete, so the application will read from the buffer and mark the data as complete like normal.
By including disk replication in the checkpoints this way, disk integrity is preserved even across system crashes.


## Consistency-breaking Execution

Like I mentioned above, network I/O passes through a replicated buffer.
If there is any incoming network traffic that arrived after the last checkpoint was taken, if the primary system crashes, then it's true that that incoming data is lost.
However, it's impossible for the network buffer to be in an invalid state, because it won't be marked as completed until after the data has been successfully committed, given to the system, and recorded in a snapshot.
Therefore, it's impossible for a newly-activated system to be in an invalid state, where it received only a partial amount of data, without that memory being lost from the buffer, because it can't be marked as completed until after the data has been completely given to the system.
When the backup system gets activated, all in-progress operations from the buffer are completed and the system moves forward in a valid state.
The same is true for disk operations.

However, Remus is not able to completely prevent duplication of outgoing network data and disk writes.
For disk writes in particular, it's okay if data from the buffer was partially written, because the data from the buffer can be copied to the disk again, (redundantly) overwriting the already-copied bytes.
For outgoing network data, though, it's impossible to "take back" incomplete data that had been partially sent before the system crashes.
In this circumstance, the newly-activated backup will send the entire in-progress block of data from the buffer, which may include some duplicate bytes.
Luckily, TCP packets are specifically designed to handle inconsistent network traffic: the data will be sent in packets with the same TCP tags, and so the client system will be able to recognize that the duplicated data is indeed duplicated, and not new information, so it becomes a non-issue.
For other network protocols, though, such as UDP or protocols designed for video game servers, duplicated data could present a problem.


# Disk Crash Consistency

Like I explained in detail above, disk operations are written to the buffer instead of to the disk directly, and once the data is ready to be committed, it's marked as in-progress and the buffer-to-disk or buffer-to-system operation starts.
Once the data is completely written to the disk or provided to the virtual machine, then the operation in the buffer is marked as complete.
The buffer is synchronized between the active and passive systems in the same way that the system snapshots are synchronized, so if the passive system gets activated (when the active system crashes), then the in-progress disk operations in the buffer are committed and marked as complete.
In this manner, disk consistency is preserved, even after a crash.

