# High-level Approach
The implementation steps are as followings:
1. Basic stop-and-wait transmission
2. Sliding window  
    Improved to the sliding window protocol, supporting the window size >= 1, 
    so that the sender can continue to send data while waiting for an ACK, increasing throughput
3. Handling out-of-order packets  
    A buffer is implemented in the receiver to store and output packets arriving in disordered.
4. Timeout and Retransmission  
    The sender maintains a dictionary of unacknowledged packets, and if a packet does not receive
    an ACK message within a certain time period, the packet with corresponding sequence will be retransmitted.
5. Detecting and handling duplicate packets  
    The receiver maintains a set to store the packets receive and output to detect duplicate packets and discard.
6. Detecting and handling corrupted packets  
    Checksum is implemented in both sender and receiver to verify the corrupted packets to discard and trigger retransmission.
7. Dynamic RTT and timeout setting
    The timestamp to record the time of message sending is implemented in receiver, so that the sender can calculate the RTT by subtraction
    the timestamp of sending message and ack message. Also, the fluctuation is considered in the timeout calculation.
8. Adaptive window size adjustment  
    The congestion control is implemented to adjust window size by using the concept of slow start
---
# Challenges
During the implementation, I encountered the several challenges and resolved them through debugging.
### Issue with sliding window and ACK handling.
**Initial Code**
```python
...
socks = select.select(sockets, [], [], self.timeout)[0]
for conn in socks:
    while len(self.unack_data) >= self.window_size:
        if conn == self.socket:
            data = self.recv(conn)
        else
            conn == sys.stdin:
...
```
**Issue**  
At initial, the code of checking the window size was written at the start of receiving from the socket. 
Once the ACK window was full, the sender could not receive new messages, causing the program to hang and preventing it from exiting.  

**Solution**  
The window control logic was restructured to ensure that even when the window was full. 
The correct logic is to only check if the window is full before sending the new packet instead of checking before receiving message from channel.

### Issue with computing checksum to detect corrupted packet
**Initial Code**
```python
    def send(self, message):
        message["checksum"] = self.compute_checksum(message)

        self.log("Sending message '%s'" % json.dumps(message))
        self.socket.sendto(json.dumps(message).encode("utf-8"), (self.host, self.port))
```
**Issue**  
Every packet cannot pass the checksum verification.

**Debugging Process**  
Added a log in send method to check if the checksum was correctly calculated
```python
self.log(f"[DEBUG] Computing checksum for message: {json.dumps(message, sort_keys=True)}")
computed_checksum = self.compute_checksum(message)
self.log(f"[DEBUG] Computed checksum: {computed_checksum}, message: {json.dumps(message, sort_keys=True)}")
```
**Solution**   
Before retransmitting, remove the old checksum field and recalculate it
```python
        if "checksum" in message:
            del message["checksum"]

        message["checksum"] = self.compute_checksum(message)
```
---
# Properties & Features of My Design
1. Reliability
- Packets are delivered in order.
- Error detection: Checksum validation ensures that corrupted packets are discarded and retransmitted.
2. Efficiency
- Dynamic RTT estimation: The sender adjusts timeout values not only based on the network's RTT but also the RTT deviation to calculate more stable timeout.
---
# Testing Overview
run the configs of different levels
> ./run configs/1-1-basic.conf  

Use logging to narrow down the problem code chunk

**Debugging Example**
```python
self.log(f"[DEBUG]Unacknowledged packets: {list(self.unacked_packets.keys())}")
```
```python
self.log(f"Checksum computed: {computed_checksum}, Expected: {msg['checksum']}")
```
...  

**Resolved Bugs**
- Fixed the sliding window ACK issue (See Challenge 1).
- Fixed checksum calculation error (See Challenge 2).
- Fixed the low bandwidth test (overhead) by reducing the initial window size
