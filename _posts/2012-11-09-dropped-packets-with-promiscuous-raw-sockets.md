---
layout: post
title: Dropped packets with promiscuous raw sockets and winsock
date: 2012-11-09 17:34:56.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags:
- c#
- Sockets
- TCP
meta:
  _edit_last: '1'
  dsq_thread_id: '900402478'
  _syntaxhighlighter_encoded: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561125259;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4737;}i:1;a:1:{s:2:"id";i:1587;}i:2;a:1:{s:2:"id";i:4286;}}}}

permalink: "/2012/11/09/dropped-packets-with-promiscuous-raw-sockets/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/dropped-packets-with-promiscuous-raw-sockets/)_

Lately in my spare time, I've been working on a tool that will decode serialized AMF over a tcp connection. AMF stands for [action message format](http://en.wikipedia.org/wiki/Action_Message_Format) and is used to serialize binary data to actionscript applications. The idea is to have the tool work the way [Charles](http://www.charlesproxy.com/) does for JSON/AMF over http/https, except over TCP sockets. I really like the way Charles works, and it'd be nice to not have to go to [Wireshark](http://www.wireshark.org/) and try and piece through binary data when I'm debugging.

So how would I do this? TCP sockets are connection oriented, you connect to some host and port and you only recieve and send data to that port. That's great and all, but you can't always inject yourself as a proxy in a connection; it'd be nice to be able to just sit in the middle of a conversation and observe without interfering. Thankfully you actually can do this by creating a [raw](http://en.wikipedia.org/wiki/Raw_socket) [promiscuous](http://en.wikipedia.org/wiki/Promiscuous_mode) socket which captures all information regardless of port. This lets you inspect data like ip headers and tcp/udp/icmp/etc headers of all packets going through your network card (regardless if they are even for you!).

Raw sockets work the way they sound; they gives you the raw information including IP headers and other protocol headers (depending on which mode you set the socket in). Promiscuous mode tells your network card to not filter packets based on port or IP, just to give you everything. This way you can inspect all the packets going through your machine. For my project, a [coworker](http://tech.blinemedical.com/author/faisal-mansoor/) suggested I use [WinPcap](http://www.WinPcap.org/) but I didn't want to create a hard dependency (you need to install a driver) to it for what I thought would be some basic c++ so I started off with just raw sockets.

Initially, this worked great. I was able to re-assemble fragmented tcp packets, inspect IP/TCP headers, correlate data by source/destination port, etc. pretty easily. But I noticed that sometimes when I was reassembling a large packet that was fragmented (greater than the [MTU](http://en.wikipedia.org/wiki/Maximum_transmission_unit)), I wouldn't get all of the packets. I fired up Wireshark and compared my results. It was pretty clear that I was missing almost half of the remaining packets. When the data sent was less than the MTU, I got it just fine. Clearly something was being dropped, and I wasn't [the only one](http://developerweb.net/viewtopic.php?id=5885) who had this problem.

Maybe I just wasn't reading fast enough? I commented out all code other than directly reading off the socket and just printed the size of the packets I got. Even then, I still wasn't matching up to what Wireshark had.

Here was a stripped down version of my basic capture code for reference, nothing fancy. Just reading off a socket created with `SOCK_RAW`

[c language="++"]  
#define High4Bits(x) ((x\>\>4) & 0x0F)

void Run()  
{  
 sockaddr\_in socketDefinition;

socketPtr = socket( AF\_INET, SOCK\_RAW, IPPROTO\_IP );

BindSocketToIp(socketPtr, socketDefinition);

CreatePromisciousSocket(socketPtr);

while(true){  
 int bytesRead = recv( socketPtr, packet, LS\_MAX\_PACKET\_SIZE, 0 );

if ( bytesRead-\>ver\_ihl) != 4 ){  
 delete packet;  
 return;  
 }

ipHeaderSize = Low4Bits(ipHeader-\>ver\_ihl);  
 ipHeaderSize \*= sizeof(DWORD);

switch( ipHeader-\>protocol )  
 {  
 case 6: // TCP  
 {  
 char \* tcpHeaderStart = &packet[ipHeaderSize];

if(TargetPortFound(tcpHeaderStart)){

printf("Got tcp/ip packet size %dn", bytesRead);  
 }

break;  
 }  
 }  
 }  
}

void CreatePromisciousSocket(SOCKET socketPtr){  
 int optval = 1;  
 DWORD dwLen = 0;

if ( WSAIoctl( socketPtr,  
 SIO\_RCVALL,  
 &optval,  
 sizeof(optval),  
 NULL,  
 0,  
 &dwLen,  
 NULL,  
 NULL ) == SOCKET\_ERROR )

{  
 printf( "Error setting promiscious mode: WSAIoctl = %ldn", WSAGetLastError() );  
 throw "Error setting promsocous mode";  
 }  
}

void BindSocketToIp(SOCKET socketPtr, sockaddr\_in socketDefinition){  
 char localIp[20] = "192.168.1.2";

socketDefinition.sin\_family = AF\_INET;

socketDefinition.sin\_addr.s\_addr = inet\_addr(localIp);

if ( bind( socketPtr, (struct sockaddr \*)&socketDefinition, sizeof(socketDefinition) ) == SOCKET\_ERROR )  
 {  
 printf( "Error: bind = %ldn", WSAGetLastError() );  
 throw "Error binding";  
 }  
}

bool TargetPortFound(char \*packet, int targetPort)  
{  
 TCPHEADER \*tcp\_header = (TCPHEADER \*)packet;

if(htons(tcp\_header-\>source\_port) == targetPort){  
 return true;  
 }

return false;  
}

[/c]

The side by side comparison of filtering on port 21935 was:

[![](http://tech.blinemedical.com/wp-content/uploads/2012/11/wireSharkPacketsBroken.-300x151.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/11/wireSharkPacketsBroken..jpg)

Bear with me while I explain what we're looking at here. This is a capture of a 50k AMF response simultaneously with Wireshark (on the right) and my program using raw sockets (on the left). Both sides should have had about 40 packets come through, but you can see that my program on the left is missing a bunch of the larger packets compared to Wireshark on the right. After the 40 packets came through a small 38 byte AMF message was sent and you can see that both my program and Wireshark got the packet. Somehow a bunch of packets went missing for me! Don't be confused by the different numbers. On the left hand side (my program using raw sockets), the size includes IP header, TCP header, and payload length. On the right, in Wireshark, I've highlighted JUST the data payload. So the highlighted area on the right is 40 bytes LESS than the highlighted area on the left (IP header is 20 bytes, and TCP header is 20 bytes). So if you see 78 on the left, that's really a 38 byte payload plus 40 bytes of header. This can make things a little confusing, but it all matches up.

[TangentSoft's](http://tangentsoft.net/wskfaq/advanced.html) advanced winsock FAQ tipped me off to the actual problem:

> Most other common desktop operating systems have some way to ask the kernel to do some of the filtering for you. Not so with SIO\_RCVALL. You want this, because your program is probably interested in only some packets, so you have to filter out the ones you aren’t interested in. At gigabit speeds, it can take a surprising amount of CPU power to do this. **You might not be able to do it fast enough to prevent the kernel from running out of buffer space, forcing it to drop packets**. Doing at least some of the filtering in the kernel can make this practical, since it saves a kernel to user space context switch for each filtered packet.

It turns out the kernel couldn't buffer enough information for me. The raw socket was giving me all the packets going through my NIC, not just the ports that I wanted and each packet that I got required a kernel to user space context switch. Just watch Wireshark with just a tcp filter and see how much traffic is going through, it's more than you think.

The socket's [default buffer](http://stackoverflow.com/a/1507551/310196) size is only 8k so if I am making a single call that is sending 50k of AMF then that can easily bump out other packets and also get bumped out itself! While I could've [increased the buffer size](http://msdn.microsoft.com/en-us/library/windows/hardware/ff570832(v=vs.85).aspx), the documentation says it's only available on Windows Vista and later, and only some protocols (like TCP) support it. If I ever wanted to expand my tool to use other protocols then I could face this issue again. On top of that, in researching about the socket buffer size I found this quote from an old [windows socket's programming](http://www.sockets.com/ch16.htm) book:

> **You can (and should) avoid dependence on some optional features by redesigning your application**. For example, you shouldn't require a specific amount of receive buffer space for your application to function. This **doesn't require WinSocks to support the SO\_RCVBUF socket option** , so you may not be able to specify the system buffer space you get.

Through the course of my research I'd uncovered a whole slew of negative reasons to use raw sockets other than the dropped packets. Raw sockets aren't supported on all Windows versions ([like Windows XP](http://seclists.org/nmap-hackers/2005/4)) and you had to be an admin to run an application using raw sockets. This means distribution of this app or using it on a client to debug could be problematic.

At this point I was frustrated enough to scrap my raw sockets idea and switch to WinPcap. WinPcap works lower in the networking stack than a raw socket does. Raw sockets work at level 3 (network layer) but WinPcap and its associated driver sit at level 2, the data link layer. Just as a reminder, here is the ubiqutous "TCP/IP Stack" (image taken from [tcpipguide.com](http://www.tcpipguide.com/free/t_TCPIPArchitectureandtheTCPIPModel-2.htm))

[![](http://tech.blinemedical.com/wp-content/uploads/2012/10/tcpiplayers-300x257.png)](http://tech.blinemedical.com/wp-content/uploads/2012/10/tcpiplayers.png)

The real power of WinPcap is the kernel-level filtering it can do based on filter text you pass it, alleviating you costly context switches. Remember that Wireshark filter you always put in? This is what it does.

[![](http://tech.blinemedical.com/wp-content/uploads/2012/10/wiresharkfilter..png)](http://tech.blinemedical.com/wp-content/uploads/2012/10/wiresharkfilter..png)

Once I switched over to WinPcap instead of just pure raw sockets I started getting all my data without having to increase any buffer sizes. Thankfully the WinPcap [examples](http://www.winpcap.org/docs/docs_412/html/group __wpcap__ tut.html) are well documented, and it's a pretty close drop-in for raw sockets anyways, so the amount of work to switch over was pretty minimal.

Here are the new side by side screenshots of my capture application vs Wireshark. This time, all the packets are there. The data sizes I've highlighted match up this time because with WinPcap I'm actually getting the ethernet frame header, the ip header, AND the tcp header. This matches with the "length" field in Wireshark.

[![](http://tech.blinemedical.com/wp-content/uploads/2012/10/wireSharkPackets-300x140.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/10/wireSharkPackets.jpg)

After I had done all this socket research, and already changed my code to use WinPCap, I went back and increased the buffer size as a test and I was able to finally get all the packets I wanted. In the end all I really needed was the following snippet after I had bound my socket.

[c]  
int bufferLength;  
int bufferLengthPtrSize = sizeof(int);

getsockopt(socketPtr, SOL\_SOCKET, SO\_RCVBUF, (char \*)&bufferLength, &bufferLengthPtrSize);

printf("default socket buffer bytes %dn", bufferLength);

int buffsize = 50000;

setsockopt(socketPtr, SOL\_SOCKET, SO\_RCVBUF, (char \*)&buffsize, sizeof(buffsize));

getsockopt(socketPtr, SOL\_SOCKET, SO\_RCVBUF, (char \*)&bufferLength, &bufferLengthPtrSize);

printf("updated socket buffer bytes %dn", bufferLength);  
[/c]

This prints out:

[c]  
default socket buffer bytes 8192  
updated socket buffer bytes 50000  
[/c]

Though while this did work, I'm glad I went with WinPCap since depending on socket throughput I might still have run into issues.

