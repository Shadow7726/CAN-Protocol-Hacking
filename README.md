<p align="center">
  <img  width="180" src="ghost.png" />
</p>

<h1 align="center"> <b>Can Hacking</b></h1>
<h3 align="center"><b>CAN Protocol HACKING Checklist</b></h3> 

### CAN Hacking TOOLS list :

* can-utils
* Installing CAN Utils
* [Complete Can Hacking tools](https://github.com/iDoka/awesome-canbus)
* [SocketCAN userspace utilities and tools](https://github.com/linux-can/can-utils)
* [CAN-BUS_Shield_V2](https://wiki.dfrobot.com/CAN-BUS_Shield_V2__SKU__DFR0370_)

***

#### can-utils is a Linux specific set of utilities that enables Linux to communicate with the CAN network on the vehicle. The canutils consist of 5 main tools that we use very frequently:
- cansniffer for sniffing the packets
- cansend for writing a packet
- candump dump all received packets
- canplayer to replay CAN packets
- cangen to generate random CAN packets
 + **can-utils can be installed via apt-get**
 ```
 sudo apt-get install can-utils -y
 ```
 ***
 
 + **Installation of CANghost :**
 ```
 git clone https://github.com/souravbaghz/CANghost
 ```
 + **Usage:**
 ```
 cd CANghost/
 bash canghost.sh <interface> <logfilename>
 Example- bash canghost.sh vcan0 demolog
 ```
 
  + **Native CAN interface:**
 ``` 
 $ sudo ip link set can0 type can bitrate 125000
 $ sudo ip link set up can0
 ```
  
 + **Virtual CAN interface:**
  ```
 $ modprobe vcan
 $ sudo ip link add dev vcan0 type vcan
 $ sudo ip link set up vcan0
 ```
 
 
 <p align="center">
  <img  width="900" src="Screenshot.png" />
</p>

 ***
 <h3 align="center"><b>A Quick Tutorial</b></h3> 
 
 + **can dump:**
 ```
 candump -l can0 
 wc -l abcd.log [to see the number of lines using word count]
 split -l 30000 abcd.log frame1 [to split the can data- rename the file]
 
 candump -l can0 [it will log and saved in the root directory]

 cat abcd.log [To view the data]

 ```
 
  **Fuzzing using can sniffer:**
 * CAN sniffer
 ```
 cansniffer [can interface]
 example: cansniffer can0
 cansniffer -c can0 [highlights the changes in the red color] [it will show changes when we perform certain operation]
 ```
  **Filter by ID:**
 ```
 cansniffer -c can0
 "-000000" Enter buton [clear the screen]
* if you want to monitor specific id
 ex: 65c

 +0x65c enter button [if the frame is changing it will show and if it is static it will clear]
 ```
 
 + **CAN Replay attack:**
 ```
 man canplayer
 
 canplayer -I abcd.log
 ```
 
  + **CAN send utility:**
 ```
 man cansend
 ```
 
   + **Example of Replay attack:**
 ```
 candump -c -l -s 0 -a can0 [logging the can frame]
 canplayer -I abcd.log [sending the logged can frame file]
 ```
 
 
