# Lab 4 - OS & Network Substrate using OSI layers to Trace & Debug
This exercise mainly focus devops terrain and role of a DevOps engineer within the OSI ( Open System Interconnection) Layer, which consists of 7 main layer:

Physical - L1 Copper ,fiber Is the light blinking? Amber, Red or Green
Datalink - L2 `ip neigh` Ethernet/ARP
Networking - L3 `ip route` `traceroute` IP.ICMP,BGP
Transport - L4 `ss -tupln` 
Session - L5 Stick Sessons (cookies)
Presentation - L6 TLS/json, Cert chain `openssl s_client -connect example.com:443 -showcerts`
Application - L7 `curl` HTTP/gRPC status code 200-500+ *Dns operates here*

*Current Workspace* - Running Ubuntu on Orbstack

### Task 1 - Trace Request E2E

*Start QuickNotes (Terminal A)*
- Started by spliting terminal into 3 way view for each subtasks.
- On terminal one change directory to `app/` where go app resides before  Launching *quicknotes* by running `go run .`

*Capture Packet trace with tcpdump (Terminal B)*
- `tcpdump` starts capture in interactive mode using the loopback address of host machine with app listening port 8080 and Process ID while generating its encoded output to `lab4-trace.pcap` -> `sudo tcpdump -i lo -nn -s 0 -A 'tcp port 8080' -w lab4-trace.pcap &` in the background and set ENV  `TCPDUMP_PID=$!`, This capture the entire TCP transaction e2e.

*Send POST action to quicknotes (Terminal C)*
```bash
curl -v -X POST http://localhost:8080/notes \
  -H 'Content-Type: application/json' \
  -d '{"title":"trace me","body":"in flight"}'
```

*Stop Capture*
```bash
sudo kill $TCPDUMP_PID
wait $TCPDUMP_PID 2>/dev/null
```

*1.2 - Decode the encoded tcpdump capture by reading into Human Readable txt output*
`sudo tcpdump -r lab4-trace.pcap -nn -A | tee lab4-trace.txt`

#### HTTP Transaction Anatomy over TCP ( SYN -> SYN/ACK <- ACK)
*Source*                               *Destination*
Client (::1:52460)                     Server (::1:8080)

SYN[S]        --------------------------> 
           <----------------------    SYN-ACK[S.]
ACK[.]        -------------------------->

POST /notes[P.] ------------------------->
           <----------------------    ACK[.]

           <----------------------    HTTP/1.1 201 Created[P.]

ACK[.]        -------------------------->

FIN[F]        -------------------------->
           <----------------------    FIN[F]

ACK[.]        -------------------------->

*Output Flow*
- curl on ::1:52460 opens a TCP connection to ::1:8080.
- Three-way handshake completes (SYN → SYN/ACK → ACK).
- curl sends a 175-byte HTTP POST containing JSON.
- Server ACKs the request.
- Server creates a note (id=7).
- Server returns HTTP/1.1 201 Created with a JSON response.
- Client ACKs the response.
- Client gracefully closes with FIN.
- Server ACKs and sends its own FIN.
- Client ACKs the server FIN.
- TCP connection terminates cleanly.

```bash
14:13:35.038508 IP6 ::1.52460 > ::1.8080: Flags [S], seq 4083792821, win 65476, options [mss 65476,sackOK,TS val 1845403447 ecr 0,nop,wscale 9], length 0 #SYN
14:13:35.038526 IP6 ::1.8080 > ::1.52460: Flags [S.], seq 658168439, ack 4083792822, win 65464, options [mss 65476,sackOK,TS val 2990999919 ecr 1845403447,nop,wscale 9], length 0 #SYN+ACK
14:13:35.038533 IP6 ::1.52460 > ::1.8080: Flags [.], ack 1, win 128, options [nop,nop,TS val 1845403447 ecr 2990999919], length 0 #ACK
14:13:35.038584 IP6 ::1.52460 > ::1.8080: Flags [P.], seq 1:176, ack 1, win 128, options [nop,nop,TS val 1845403447 ecr 2990999919], length 175: HTTP: POST /notes HTTP/1.1 #PUSH+ACK
14:13:35.038586 IP6 ::1.8080 > ::1.52460: Flags [.], ack 176, win 128, options [nop,nop,TS val 2990999919 ecr 1845403447], length 0 #ACK
14:13:35.040440 IP6 ::1.8080 > ::1.52460: Flags [P.], seq 1:207, ack 176, win 128, options [nop,nop,TS val 2990999921 ecr 1845403447], length 206: HTTP: HTTP/1.1 201 Created #PUSH+ACK
14:13:35.040445 IP6 ::1.52460 > ::1.8080: Flags [.], ack 207, win 128, options [nop,nop,TS val 1845403449 ecr 2990999921], length 0 #ACK
14:13:35.040530 IP6 ::1.52460 > ::1.8080: Flags [F.], seq 176, ack 207, win 128, options [nop,nop,TS val 1845403449 ecr 2990999921], length 0 #FINISH+ACK
14:13:35.040552 IP6 ::1.8080 > ::1.52460: Flags [F.], seq 207, ack 177, win 128, options [nop,nop,TS val 2990999921 ecr 1845403449], length 0 #FINISH+ACK
14:13:35.040566 IP6 ::1.52460 > ::1.8080: Flags [.], ack 208, win 128, options [nop,nop,TS val 1845403449 ecr 2990999921], length 0 #ACK
```

#### Debugging

```bash
#Listening Port
ss -tlnp | grep :8080
LISTEN 0      4096               *:8080             *:*    users:(("quicknotes",pid=39999,fd=3))   
#Routes
ip route show
default via 192.168.139.1 dev eth0 proto dhcp src 192.168.139.85 metric 100 
0.250.250.200 via 192.168.139.1 dev eth0 proto dhcp src 192.168.139.85 metric 100 
192.168.139.0/24 dev eth0 proto kernel scope link src 192.168.139.85 metric 100 
192.168.139.1 dev eth0 proto dhcp scope link src 192.168.139.85 metric 100 
#Reachability
mtr -rwc 5 localhost
Start: 2026-11-11T14:34:44+0000
HOST: poc-lab   Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- localhost  0.0%     5    0.1   0.1   0.1   0.2   0.1
#Dig
dig +short example.com @1.1.1.1
172.66.147.243
104.20.23.154
#Journal
journalctl --user -u quicknotes -n 20 || true
-- No entries --
```

*Bad Gateway(502) error approach*
Firstly, Its important to understand the error code class for a strategic approach, `5xx` indicates a server side error especially with `502` which points to a bad gateway:
- query endpoint `curl -v https://localhost:8080/quicknotes/health
- Confirm app is running `systemctl status <service>`
- Check listening port `ss -tlnp | grep <port>`