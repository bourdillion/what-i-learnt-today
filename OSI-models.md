## OSI Models

**What I learnt today - 02/12/2025**

The OSI model (Open Systems Interconnection model) is a conceptual framework used to understand and standardize how different networking systems communicate over a network.

There are different level or layers of OSI, and each of this layers have group of application at their different levels.

1. **Application layers:** This standardizes how different application in different OS works, it include protocols like Http/s for web rendering, smtp for messaging, ftp for file transfer etc.
   
2. **Presentation layer:** This does 3 major stuff - a. Translation - mainly for translating ASCII to EBCDIC b. compression - either in the lossless or loosy way c. Encryption/decryption.

3. **Session layer:** Helps in managing sessions or connection between application, helps in the starting, maintaining and terminating of sessions. used in authentication and authorization.

4. **Transport layer:** Ensure reliable data transfers between application, handles error recovery and flow control. it involves 3 main steps; a. segmentation - here data received from the session layers are broken down into packets with each having a packet number and a sequence number. b. flow control - mainly syncing data movement c. error control. Examples of transport layer protocols include connection-oriented Transmission control protocol(TCP)  and connectionless User Datagram Protocol(UDP). UDP is faster than TCP because UDP does not provide any feedback.

5. **Network layer:** Determines how data moves between devices, and across multiple network. it does three major stuff: a. Logical addressing b. Routing - how data moves, it depends on ip address gotten from stage a, and mask c. path determination - network protocol determine the best path.

6. **Data link layer:** Receives IP packets from the network layer, here physical addressing is done to the data packet to form a frame, e.g of physical address is a mac address of each devices. local connection between two computers happens via the help of mac address.

7. **Physical layer:** converts the binary sequence received via data link layer to signals, which is used by the application layer



