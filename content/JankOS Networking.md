The `1,000,000x` developer [Andrew Li](https://github.com/Andwerpz) has written the best operating system ever, [jank-os](https://github.com/Andwerpz/jank-os) which is built on the best programming language, [jank-pl](https://github.com/Andwerpz/jank-pl). After bailing out on the filesystem implementation, I have decided to work on networking for the OS. I will now loosely document what I remember doing to get networking somewhat working.
# Groundwork
Developing code for the operating system has been significantly improved since I last worked on it. This made my life very easy. Mr. Li has already implemented everything I needed to start on this networking driver. You can reference his writeup to see what he has done. [JankOS](https://andwerpz.github.io/html/blogs/jank_os/jank_os.html)

For networking, I created some structs that will com in handy later.
```c
struct NetInterface {
    u8* name;

    // L2
    u8[6] mac_address;
    
    // L3
    u32 ip_address;
    u32 subnet_mask;
    u32 gateway_ip;
    u32 dhcp_server;
    u32 dns_server;
    i32 dhcp_configured;

    // driver abstractions
    fn<void(u8*, u16)> send;
    fn<i32()> get_link_state;
}
```
# Objective
The goal was for me to write a somewhat functional network stack for user mode programs. The loose order of steps were to:
1. Write the driver for the specific NIC (network interface card)
2. Implement the protocols as needed
To be more precise, I needed to implement these protocols for bare minimum:
- Ethernet
- ARP
- IP
- UDP
- DHCP
These other protocols were added in either for testing or to get more functionality:
- ICMP
- TCP
# NIC Driver
Initially, I was only going to support the Intel e1000 network card, since it was well documented and QEMU was able to virtualize it. I later went and wrote the driver for the Realtek RTL8111 drivers to test on my old laptop.
## e1000
I mainly reference the [OSDev Wiki](https://wiki.osdev.org/Intel_Ethernet_i217) and the [Intel Developer Manual](https://github.com/Andwerpz/jank-os/blob/c5f149dec534e623c10d0e0166a5ee318cf38efd/docs/e1000.pdf). 
*Note: The OSDev code has both Memory Mapped IO (MMIO) and Port Mapped IO (PMIO) in their examples. I chose to just use MMIO.*
**Since I am using DMA and MMIO, I needed to make sure that the memory is set to uncacheable when mapping it*
### Basic Initialization
#### PCI
After detecting the card in ACPI, I had to setup the PCI configuration for the card. This composed of the following steps:
1. Detecting the Base Address Register (BAR) type and memory mapping it.
2. Toggling certain features in the PCI command register:
	1. Bus Mastering for Direct Memory Access (DMA)
	2. Memory Access Enable to signal that its memory mapped
3. Setup interrupts:
	1. Grab the Interrupt Request (IRQ) line from the PCI header
	2. Calculate the interrupt vector (We used an offset of 30)
#### EPROM
Reading the EEPROM for the MAC address.
The EEPROM read has a 32 bit register (`EERD` register) that I can set the read address and start bit to get information out of it.
*For the QEMU virtualized e1000 card, it does contain an EEPROM, my implementation assumes that one exists in order to extract the MAC. If one does not exist I believe there is another register that will contain the MAC.*
#### Reset
The Intel manual details the steps needed to reset/initialize the card, I just followed their instructions.
1. Disabling flow control (Setting registers `FCAL`, `FCAH`, `FCT`, `FCTTV` to 0)
2. Setting and clearing some bits in the command register:
	1. Enable auto speed detection (`ASDE`)
	2. Enable set link up (`SLU`)
	3. Disable VLANs (`VME`)
	4. Clear link reset (`LRST`)
	5. Clear PHY reset (`PHY RST`)
3. Zeroing out the Multicast Table Array (`MTA`)
### Enabling Interrupts
From the PCI setup earlier, I should have the calculated interrupt vector from the IRQ line, I can then use that to register the vector inside the Interrupt Descriptor Table (IDT), where I will handle the interrupt and eventually call the interrupt handler for the e1000 card.

1. Register the IDT entry
2. Register the IRQ line into the Programmable Interrupt Controller (PIC) by clearing the bit
3. Disable all interrupts using the `IMC` register
4. Clear all pending interrupts using the `ICR` register
5. Configure the interrupt mask using the `IMS` register:
	1. Enable Link Status Change
	2. Enable Receive Timer Interrupt

*Note: I have only enable two interrupt causes since I just wanted bare minimum functionality, all I wanted to know is when the link state changes (physical ethernet is plugged in and negotiated vs unplugged) and when a frame is received by the hardware.*
### Interrupt Handler
The interrupt handle only checks for 2 things:
1. Link state
	1. If it is down, reset the DHCP configuration
	2. If it is up, check if DHCP is configured, if not, send a DHCP Discover out
2. Receive status
	1. If frame received, forward it to the Ethernet handler
### Receive and Transmit Initialization
For receive and transmit, the hardware uses a ring buffer to store incoming and outgoing frames. For each descriptor ring, it will have its own head and tail pointer. The CPU writes to the tail and the hardware will write to the head.

I need to allocate physical memory for the hardware to access, for JankOS, I can get an identity mapped page using `palloc()`.

Here are the RX and TX descriptor structs for the e1000.
```c
struct E1000_RDESC {
    u64 addr;
    u16 length;
    u16 checksum;
    u8  status;
    u8  errors;
    u16 special;
}

struct E1000_TDESC {
    u64 addr;
    u16 length;
    u8  cso;
    u8  cmd;
    u8  status;
    u8  css;
    u16 special;
}
```

1. `palloc()` a page for the ring base, this will be an array of RX or TX descriptor structs
2. For each descriptor:
	1. `palloc()` a page for the data buffer and set `addr` to the page
	2. Set the `status` *1 for CPU control (TX default) and 0 for hardware control (RX default)*
	3. For TX, I also needed to clear `cmd`
3. Write the address of the ring descriptor to the appropriate register (`RDBAL`, `RDBAH`, `TDBAL`. `TDBAH`)
4. Write the length of the ring to the appropriate register (`RDLEN`, `TDLEN`)
5. Initialize the head and tail to 0 (`RDH`, `RDT`, `TDH`, `TDT`)
6. Enable the RX/TX:
	1. For RX:
		1. Set the proper bits in the Receive Control Register (`RCTL`):
			1. Receiver Enable (`EN`)
			2. Multicast Promiscuous Enabled (`MPE`)
			3. Broadcast Accept Mode (`BAM`)
			4. Strip Ethernet CRC from incoming packet (`SECRC`)
			5. Set the appropriate Receive Buffer Size (`BSIZE`, `BSEX`)
	2. For TX:
		1. Set the proper bits in the Transmit Control Register (`TCTL`)
			1. Transmit Enable (`EN`)
			2. Pad Short Packets (`PSP`)
### Receive
1. The CPU checks the current tail pointer
2. Busy waits until the status bit is set to Descriptor Done (`DD`)
3. Reads in the length and the buffer address
4. Passes it into the Ethernet handler
5. Update the status bit to return control to the hardware
6. Write to the tail pointer (points to the last finished one)
7. Increment the internal pointer
### Send
1. Get the current tail pointer from the Transmit Descriptor Tail (`TDT`)
2. Copy over the data into the descriptor buffer (`addr`)
3. Set the length in the descriptor (`length`)
4. Set the command bits:
	1. End of Packet (`EOP`) *Note: MTU is default 1500, so each buffer should fit inside the limits of a 4Kib page*)
	2. Insert FCS (`IFCS`)
5. Update the status to give control to the hardware
6. Increment the tail pointer
## RTL8111
The setup for this card a little more involved, but flows almost exactly like the e1000.
### Basic Initialization
#### PCI
ACPI detection is the same as the e1000, although some configuration details differ here.
1. Detecting the Base Address Register (BAR) type and memory mapping it.
2. Toggling certain features in the PCI command register:
	1. Bus Mastering for Direct Memory Access (DMA)
	2. Memory Access Enable to signal that its memory mapped
	3. Disabling Legacy interrupts
3. Enabling MSI
*The e1000 did not support MSI or MSI-X, which forces me to read the IRQ line from the PCI header, and register it into the PIC. For the RTL8111, I can utilize MSI, which allows me to set my own interrupt vector, using the APIC (Advanced Programmable Interrupt Controller). I did have to disable legacy interrupts for this to work.*
#### Reset
For resetting the device, the manual outlined a software reset using the Command Register (`CR`) and also a PHY reset through the PHY Access Register (`PHYAR`).
For both resets, I just needed to write the correct reset bit to the register, and then wait for the reset to complete by spinning until the bit flips back.
#### Obtaining Device MAC
The RTL8111 reads the MAC from the EEPROM into a register automatically on reset.
To obtain the device MAC, I just needed to read 6 bytes total from registers `IDR0` to `IDR5`.
### Receive and Transmit Initialization
Here are the RX and TX descriptor structs for the RTL8111.
*Note: opts1 and opt2 vary between descriptors, but they are described in the [manual](https://github.com/Andwerpz/jank-os/blob/8d9923afca812c40597b8a3b605dff914233a40a/docs/RTL8111B_8168B_Registers_DataSheet_1.0.pdf)*
```c
struct RTL8111_RDESC {
    u32 opts1;
    u32 opts2;
    u32 addr_lo;
    u32 addr_hi;
}

struct RTL8111_TDESC {
    u32 opts1;
    u32 opts2;
    u32 addr_lo;
    u32 addr_hi;
}
```
*Note: Filling out `opts1` and `opts2` will depend on the desired functionality of the network card. I did the bare minimum to get it working. Refer to the manual to see more options that can be set.*
1. `palloc()` a page for the ring base, this will be an array of RX or TX descriptor structs
2. For each descriptor:
	1. `palloc()` a page for the data buffer and set `addr_lo` and `addr_hi` to the page
	2. Set the size of the descriptor buffer in `opts1`
	3. For RX:
		1. We need to set the `OWN` bit in `opts1`. *When set, indicates that the descriptor is owned by the NIC, and is ready to receive a packet*
	4. Set the `EOR` (End of Ring) on the last descriptor
	5. Write the address of the ring descriptor to the appropriate register (`RDSAR`, `TNPDS`) *Note: Some legacy behavior made it so that I needed to write the high and low memory addresses separately for the transmit register.*
### Enabling Interrupts
Since I used the APIC for the interrupts, I was able to set a constant interrupt vector for this device instead of using the PIC and IRQ line from the PCI header. I registered the entry manually inside the IDT (Interrupt Descriptor Table) initialization.
1. Register the IDT entry
2. Clear all pending interrupts using the `ISR` register
3. Configure the interrupt mask using the `IMR` register:
	1. Enable Link Change Interrupt (`LinkChg`)
	2. Enable Rx OK Interrupt (`ROK`)
### Interrupt Handler
The interrupt handle only checks for 2 things:
1. Link state
	1. If it is down, reset the DHCP configuration
	2. If it is up, check if DHCP is configured, if not, send a DHCP Discover out
2. Receive status
	1. If frame received, forward it to the Ethernet handler
### Receive and Transmit Enable
1. Enable through the Command Register `CR`
2. Configure respective registers:
	1. For RX (`RCR`):
		1. Accept Broadcast Packets (`AB`)
		2. Accept Multicast Packets (`AM`)
		3. Accept Physical Match Packets (`APM`)
	2. For TX (`TCR`):
		1. Set Max DMA Burst Size to Unlimited (`MXDMA`)
		2. Set InterFrame Gap Time to default (`IFG`)
### Receive
1. The CPU retrieves the current descriptor pointer and checks to make sure the ownership bit is set to CPU
2. Read the length and the buffer address from the descriptor
3. Modify the length to remove the CRC from the frame, and pass it into the Ethernet handler
4. Return ownership of the descriptor to the hardware, and retain the `EOR` bit, clear other flags
5. Increment the current descriptor pointer

*Note: I had to subtract the CRC from the data buffer since the RTL8111 does not have a configuration to automatically truncate it. This was not a problem with the e1000 since the Intel card had an option to automatically remove the CRC from the incoming frame.*
### Send
1.  The CPU retrieves the current descriptor pointer and checks to make sure the ownership bit is set to CPU
2. Copy the buffer over into the descriptor buffer
3. Set the ownership bit to hardware
4. Set the First Segment (`FS`) and Last Segment (`LS`)
5. Preserve the `EOR`
6. Write the frame length
7. Increment the descriptor pointer
8. Write the Normal Priority bit (`NPQ`) to the Transmit Priority Polling register (`TPPoll`)

*Note: The `FS` and `LS` should the same, my current driver does not support frames that span multiple segments. For the default Ethernet MTU of 1500, there shouldn't be any issues for now.*
*Note: All frames that are sent out my driver is going to be normal priority, to do high priority, write the High Priority bit `HPQ` instead.*
# Sockets
I implemented something similar (?) to sockets in Linux. Here is the current struct for it:
```c
struct socket {
    i32 domain;
    i32 type;
    i32 protocol;
    
    proto_ops* ops;

    i32 state;          // this will be used for all types probably
    void* proto_info;   // this will be used for protocol specific stuff

    u32 local_addr;
    u16 local_port;
    u32 remote_addr;
    u16 remote_port;
    
    waitq wait_queue;
}
```

The protocol operations struct is defined as:
```c
struct proto_ops {
    fn<i32(socket*, sockaddr*, u64)> bind;
    fn<i32(socket*, i32)> listen;
    fn<socket*(socket*, sockaddr*, u64*)> accept;
    fn<i32(socket*, sockaddr*, u64)> connect;
    fn<i32(socket*, u8*, u64, i32, sockaddr*)> send;
    fn<i32(socket*, u8*, u64, i32, sockaddr*)> receive;
    fn<i32(socket*)> close;
    fn<u16(socket*)> poll;
}
```
For each protocol that is implemented, there will be a singleton protocol operations that is created when the OS is initialized.

Since a socket will be a file descriptor, I also need to initialize the file operations singleton:
```c
void init_socket_file_ops() {
    FILE_OPS_SOCKET = $file_ops* malloc(sizeof(file_ops));
    new (FILE_OPS_SOCKET) file_ops();
    FILE_OPS_SOCKET->op_read        = #<socket_read(file*, u8*, u64)>;
    FILE_OPS_SOCKET->op_write       = #<socket_write(file*, u8*, u64)>;
    FILE_OPS_SOCKET->op_lseek       = #<socket_lseek(file*, i64, i32)>;
    FILE_OPS_SOCKET->op_getdents    = #<socket_getdents(file*, u8*, u64)>;
    FILE_OPS_SOCKET->op_close       = #<socket_close(file*)>;
    FILE_OPS_SOCKET->op_truncate    = #<socket_truncate(file*, u64)>;
    FILE_OPS_SOCKET->op_fstat       = #<socket_fstat(file*, stat*)>;
    FILE_OPS_SOCKET->op_poll        = #<socket_poll(file*)>;
}
```
All generic calls like `sys_read`, `sys_write`, etc. will just call the socket protocol singleton function.
*More information about files will be on Mr. Li's writeup.*

For each protocol, I created a global port table to keep track of all the in-use ports on the OS. Here are the current port tables and their initializations.
```c
[__GLOBAL_FIRST__] u32 MAX_PORTS = $u32 (1 << 16);
[__GLOBAL_FIRST__] u32 RESERVED_PORT_START = $u32 1;
[__GLOBAL_FIRST__] u32 RESERVED_PORT_END = $u32 1024;
[__GLOBAL_FIRST__] u32 EPHEMERAL_PORT_START = $u32 49152;
[__GLOBAL_FIRST__] u32 EPHEMERAL_PORT_END = $u32 (1 << 16);

socket** UDP_PORT_TABLE;
socket** TCP_PORT_TABLE;

void init_port_tables() {
    UDP_PORT_TABLE = $socket** malloc($u64 MAX_PORTS * sizeof(socket*));
    memset($void* UDP_PORT_TABLE, 0, $u64 MAX_PORTS * sizeof(socket*));
    TCP_PORT_TABLE = $socket** malloc($u64 MAX_PORTS * sizeof(socket*));
    memset($void* TCP_PORT_TABLE, 0, $u64 MAX_PORTS * sizeof(socket*));
}
```
# Syscalls
These networking Syscalls were implemented:
- [sys_socket](https://man7.org/linux/man-pages/man0/sys_socket.h.0p.html)
- [sys_connect](https://man7.org/linux/man-pages/man2/connect.2.html)
- [sys_sendto](https://linux.die.net/man/2/sendto)
- [sys_recvfrom](https://linux.die.net/man/2/recvfrom)
- [sys_bind](https://man7.org/linux/man-pages/man2/bind.2.html)
- [sys_listen](https://man7.org/linux/man-pages/man2/listen.2.html)
- [sys_accept](https://man7.org/linux/man-pages/man2/accept.2.html)
All the syscalls implemented were made to be compatible with the syscalls on Linux (somewhat true...)

Inside each syscall:
1. Switch to kernel page table
2. Validation (Checking valid file descriptors, function parameters, etc.)
3. Copying from user to kernel
4. Calling the `socket` abstraction layer
5. Copying from kernel to user
6. Returning
This was the basic flow for all the networking syscalls.
# Protocols
## Layer 2
### Ethernet
This is the lowest protocol I needed to support in the OSI layer, all data will be eventually encapsulated with a Ethernet header and sent out.

The header only consists of 3 fields, `Destination MAC`, `Source MAC`, and `Type`.

For receiving, I just need to check the [type](https://en.wikipedia.org/wiki/EtherType) and forward it to the appropriate handler. Currently, I just have 2 types, `IP` and `ARP`.

For sending, I just need to allocate the input buffer size plus the Ethernet header size, construct the header, and copy over the input contents. After this, I can send it using the `NetInterface` send.
### ARP
ARP resolves the MAC addresses from an input IP address. This allows us to populate the Ethernet header later. 

Here are the structs for ARP operations:
```c
struct ARPHeader {
    u16     hw_type;        // Hardware Type
    u16     proto_type;     // Protocol Type
    u8      hw_len;         // Hardware Length
    u8      proto_len;      // Protocol Length
    u16     opcode;         // Operation
    u8[6]   sender_mac;     // Sender Hardware Address
    u32     sender_ip;      // Sender Protocol Address
    u8[6]   target_mac;     // Target Hardware Address
    u32     target_ip;      // Target Protocol Address
}

struct ARPEntry {
    u8[6]   mac;
    u64     expire_at;
}

struct PendingPacket {
    u32 target_ip;
    u8* data;       // this is the entire packet, NOT packet->data
    u16 len;        // this is the length of the entire packet, NOT packet->data
    PendingPacket* next;

    u32 retries;
    u64 last_attempt;
}
```

ARP requests are sent via a L2 broadcast, which means that the destination MAC of the Ethernet frame is `FF:FF:FF:FF:FF:FF`. 

A further explanation of the ARP header fields can be found on [Wikipedia](https://en.wikipedia.org/wiki/Address_Resolution_Protocol). To send a ARP request, I just needed to fill out the header and send to the buffer to be encapsulated through the Ethernet send function.

To handle ARP packets, JankOS handles two cases:
1. ARP Reply
2. ARP Request

To handle an ARP reply I just needed to parse the incoming packet, and add it to our ARP table. Our ARP table is current just a `Hashmap<IPs, Entries>` each entry will have the associated MAC address for the IP along with a expire time. Each ARP entry will expire a minute after the entry has been created.

Now, what if a packet needs to be sent to a host that is not currently in the ARP table?
Here is how JankOS handles it currently:
1. The packet metadata and its buffer gets copied into a `PendingPacket`
2. The `PendingPacket` is now placed inside a linked list
3. Whenever a new ARP reply is received, I check the linked list for if the new entry matches with the `PendingPacket`, if it does, send it out

To prevent a build up of `PendingPackets`, each one will contain a maximum amount of retries along with a timestamp of the last time the OS tried it. In the case where the IP does not exist, it should automatically be remove from the list eventually.

To handle an ARP request, I parse the incoming packet, then basically reverse the source and destination entries, filling in our information, then sending the reply back out.
## Layer 3
### IP
The current JankOS network stack only supports IPv4.

Here is the structs I used, which are copied from [Wikipedia](https://en.wikipedia.org/wiki/IPv4#Header):
```c
struct IPv4Header {
    u8  version_ihl;        // Version: 4 bits + Internet Header Length (IHL): 4 bits
    u8  tos;                // Differentiated Services Code Point (DSCP): 6 bits + Explicit Congestion Notification (ECN): 2 bits
    u16 total_length;       // Total Length: 16 bits
    u16 id;                 // Identification: 16 bits
    u16 flags_fragment;     // Flags: 3 bits + Fragment Offset: 13 bits
    u8  ttl;                // Time to live (TTL): 8 bits
    u8  protocol;           // Protocol: 8 bits
    u16 header_checksum;    // Header Checksum: 16 bits
    u32 src_ip;             // Source address: 32 bits
    u32 dst_ip;             // Destination address: 32 bits
}
```

Like Ethernet, handling incoming IP packets just relies on correctly routing the protocol. Currently, JankOS only handles 3 protocols, `ICMP`, `TCP`, and `UDP`.

Since this is on layer 3, there needs to be some checks to make sure the egress traffic is routed correctly. When the OS wants to send a packet, it first construct the IP header, along with the [checksum](https://en.wikipedia.org/wiki/Internet_checksum). After that, the checks are done:
- If the destination IP is in the `127.0.0.0/8` (loopback) subnet or matches with the current interfaces assigned IP, then it sends the packet back into the IP handler.
- If the IP matches with the broadcast address `255.255.255.255`, it sends the frame with the broadcast MAC `FF:FF:FF:FF:FF:FF`.
- If the destination address is not within the current interface subnet, then it must change the destination (or rather the next hop address) to the default gateway.
Once it has constructed a valid IP header, it needs to check if the destination IP exists inside the current ARP table.
- If it exists: Lookup the MAC in the table, and send the Ethernet frame out.
- If it does not exist: Queue the packet (from [[#ARP]] earlier), and send an ARP request
### ICMP
Current ICMP implementation only supports responding to a ping, and also sending a ping out.
Here is the header:
```c
struct ICMPHeader {
    u8  type;
    u8  code;
    u16 checksum;
    u16 identifier;
    u16 seq_num;
}
```
#### Receiving
When the OS receives an ICMP request, to respond, all it has to do is change the header type to reply, and respond back.
#### Sending
To send an ICMP request, the header needs to be filled out with the right type and code, then setting a *random(?)* identifier and sequence number. Finally, calculate the checksum (same way as IP header) and send it off. 
## Layer 4
### UDP
UDP is based on 'fire-and-forget,' which makes it easier to implement over TCP. I did not have to worry about states as UDP is stateless. Here are the structs used:
```c
struct UDPHeader {
    u16 src_port;
    u16 dst_port;
    u16 length;
    u16 checksum;
}

struct UDPSock {
    Packet* rx_head; 
    Packet* rx_tail;
}
```
*Note: the `UDPSock` struct is used to keep a linked list of packets under the `Socket` struct*
#### Send
1. Encapsulate the data with the UDP header, then send out.
#### Receive
1. Check the associated UDP Socket buffer for available packets
	1. If empty, wait using `wait_queue`
	2. If available, pop the packet and return
The default behavior blocks, so when the non-blocking flag used, it just immediately returns.
#### Bind
1. Creates an entry in the UDP port table with either the specified port, else it pulls a port from the ephemeral range
#### Connect
1. Updates the `socket` to the given address and port
Since UDP is stateless, there isn't a handshake, so I can just set the variable.
#### Listen
UDP doesn't support this, once an entry is made in the UDP port table, incoming connections will automatically get forwarded.
#### Accept
UDP doesn't support this. There are no states, thus this is not implemented.
#### Poll
1. Checks packet queue to see if there any packets available (sets `POLLIN`)
2. Always set `POLLOUT` ('fire-and-forget')
#### Close
1. Clears UDP port table entry
2. Frees all packets in queue
### TCP
TCP was slapped together very poorly. But it functions (barely) and thats probably good enough. Here are the structs used:
```c
struct TCPHeader {
    u16 src_port;
    u16 dst_port;
    u32 seq_num;
    u32 ack_num;
    u8  data_offset;
    u8  flags;
    u16 window_size;
    u16 checksum;
    u16 urgent_ptr;
}

struct TCPSock {
    i32 state;

    u32 snd_una;    // oldest ack'd
    u32 snd_nxt;    // next seq to send
    u32 snd_wnd;    // send window
    u32 iss;        // initial send seq num

    u32 rcv_nxt;    // next expected seq
    u32 rcv_wnd;    // receive window
    u32 irs;        // initial receive seq num

    u8* rx_buf;
    u32 rx_head;
    u32 rx_tail;

    i32 backlog;
    deque<socket*> accept_q;
    vector<socket*> children;
}
```
#### Send
#### Receive
#### Bind
#### Connect
#### Listen
#### Accept
#### Poll
#### Close
## Layer 7
### DHCP
DHCP needs to be ran when the network device link status changes. This allows the machine to renegotiate and obtain the IP address it will use to communicate on the network.
The machine needs to go through the DHCP DORA process below:
- Discover
- Offer
- Request
- Acknowledge
Here are the structs used:
```c
struct DHCPHeader {
    u8      op;                 // packet type
    u8      htype;              // hardware address type
    u8      hlen;               // length of hardware addr
    u8      hops;               // hops
    u32     xid;                // random transaction number
    u16     secs;               // seconds usted in timing
    u16     flags;              // flags
    u32     client_ip_addr;     // ip addr of this machine (if we alr have)
    u32     your_ip_addr;       // ip addr of this machine (offered by dhcp server)
    u32     server_ip_addr;     // dhcp server ip addr
    u32     gateway_ip_addr;    // dhcp relay ip addr
    u8[16]  client_hw_addr;     // hardware addrs
    u8[192] bootp_legacy;       // make this all 0 hehe
    u32     magic_cookie;       // need this to differentiate bootp from dhcp
}

struct DHCPDiscoverOptions {
    // option 53
    u8 opt_msg_type_code;
    u8 opt_msg_type_len;
    u8 opt_msg_type_val;

    // option 55
    u8 opt_req_list_code;
    u8 opt_req_list_len;
    u8 req_item_subnet;
    u8 req_item_router;
    u8 req_item_dns;

    // end
    u8 opt_end;
}

struct DHCPRequestOptions {
    // option 53
    u8 opt_msg_type_code;
    u8 opt_msg_type_len;
    u8 opt_msg_type_val;

    // option 50
    u8 opt_req_ip_code;
    u8 opt_req_ip_len;
    u32 opt_req_ip_val;

    // option 54
    u8 opt_server_id_code;
    u8 opt_server_id_len;
    u32 opt_server_id_val;

    // option 55
    u8 opt_req_list_code;
    u8 opt_req_list_len;
    u8 req_item_subnet;
    u8 req_item_router;
    u8 req_item_dns;

    // end
    u8 opt_end;
}
```
More details can be found on this [Wikipedia page](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol).

*Note: DHCP Options are packed using [Type-length-value](https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value)*
#### Discover
In this stage, the host machine sends a DHCP packet out to the default IPv4 broadcast address (`255.255.255.255`). To fill out the header, all constants should be configured to be set for DHCP Discover. For everything else, it should be 0, since we don't actually have the network configured. For options, I request the subnet mask, DNS server, and default gateway of the network.
#### Offer
Once the DHCP server responds, the machine has to handle the offer. The main thing to see in the offer should be the server IP and the offered IP. After parsing the offer, it will need to immediately send a request out to the server.
#### Request
Once the server offers us an IP, the machine needs to 'claim' the IP by requesting it. The header is filled and the packet is sent to the server IP.
#### Acknowledge
Once the server acknowledges the request, the machine is now in ownership of the earlier requested IP. Here is where I can parse the options fields to copy over the subnet mask, DNS server, and the default gateway to the `NetInterface` struct.
# Future Work and Improvements
A lot of work needs to be done for current and new features. Here are some that I have written down as I wrote the stack.
- Using enabling more interrupts on the NICs, such as transmit success or failures
- Polish up DHCP on link state change
- Adding lease times to DHCP
- More robust error handling
- Disabling network stack if no device is present
- Generic network driver struct
- TCP `accept()` enforcing backlog from `listen()`
- TCP out of order packet handling
- TCP congestion control
- Connection timeouts
- DNS implementation
- Generic socket states tied into TCP and UDP
- Hardware driver robustness, such as handling cases where the ring is full
# References
This is the `1,000,000x` developer.
![[millionxdeveloper.jpg]]