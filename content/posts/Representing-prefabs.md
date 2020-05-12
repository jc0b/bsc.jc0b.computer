---
title: "Representing Prefabs"
date: 2020-05-09T21:00:00+02:00
draft: false
toc: false
images:
tags:
  - prefabs
  - design
  - architecture
---

A large part of my project is based on designing a model for how we can represent datacenter topologies (and thus, prefabs) in OpenDC. This has proven to be quite a challenge to conceptualize, because there isn't one "right" way of doing so, and the many possibilities can be quite intimidating.

## A dependency model
Because we are dealing with objects, we are also dealing with object relationships. From the beginning, I felt that it would be quite sensible to make sure that each component object was either a parent object, or had children objects (or both). This is how I've always personally understood computer internals: **there are some components you just can't have without others**. With this in mind, I felt it was appropriate to model dependency relationships between components.
Because I want to make sure that, at a minimum, a prefab can execute workloads (and is therefore a functioning computer with all the components required to pass POST), a minimal prefab still has hardware requirements, or dependencies. A motherboard requires a chassis to be housed in, and a CPU requires a motherboard with the appropriate socket. I also think that servers require racks. While this isn't a hard requirement (I have had a variety of rack servers under my desk without a rack), it is definitely considered to be best practice in a data center. Following on from this train of thought, I devised the following diagram:
![Object Diagram](/images/prefab-objects.jpg)


We'll start from the top.
### Environments
Sometimes (for example, in distributed computing), you may want to model physically seperate data centers. It's also possible that your organisation has multiple rooms, each containing a different data center. I didn't think it made sense to limit design to just one data center at a time, so I started with environments. These environments contain rooms, and should be given a name.
Environments must contain **at least** one room, but can contain an infinite number of rooms.
### Rooms
Rooms enact constraints upon data centers, so their presence is important in this model. Depending on the amount of compute that is desired in a single data center, compute density may increase or decrease with the size of the room that the data center is stored in. 
Rooms also have location characteristics (perhaps a room number, or a city or country). Again, we can also name our rooms.
Rooms must contain **at least** one data center (otherwise they don't really have a purpose). They can also contain a theoretically boundless number of data centers, but in practice this will be bounded by the number of racks you can fit in a single room.
### Data centers & Racks
Data centers have names and physical locations. Their children are racks, which contain the compute hardware stored within. Racks also have a size in terms of rack units.
### Rack hardware
So far, there are 4 types of rack hardware I have determined are fairly common in racks. These are server chassis, network switching hardware, Power Delivery Units (PDUs) and Uninterruptible Power Supplies (UPSes).
#### Chassis
A chassis serves as a container for the server hardware that we need to execute workloads. Depending on the contents, a chassis may also have a size in rack units (Us)
#### Switches
Networking hardware that serves to connect individual server nodes together (and to the internet, if relevant). May not be relevant to the simulation, but may be useful to incorporate in prefabs to make the topology representation complete. For example, a rack-sized webserver prefab would likely not be particularly useful in real life without a switch
##### Power Delivery Units & Uninterruptible Power Supplies
Two more utilities as above. PDUs serve to deliver power to hardware in the rack, with UPSes serving to ensure that rack hardware isn't suddenly powered off due to a power outage. Both of these typically take up some rack units, but there are also 0U PDUs that are attached vertically to the rack that are worth mentioning. When simulating power usage, it may be useful to determine whether a system can actually be powered in its current configuration.
### Server Components
These components are the ones most relevant to simulating our workloads. Without properly populated values for these components, simulations from OpenDC would not be representative of the expected performance at a given workload.
#### Motherboard
This component is considered to be the child of the chassis - it has to be housed inside of it. This component may have a varying number of CPU sockets (up to 4 sockets seems to be common on mass-produced motherboards), as well as a varying number of memory sockets, network ports and PCI express expansion cards. We anticipate that a server motherboard will have at least one of all of the above.
#### Power Supply
This component is another child of the chassis, also being housed within it. Power supplies have several characteristics, chief among these being the rated wattage, the efficiency rating, and whether it uses Alternating Current (AC) or Direct Current (DC) for power.
In addition, multiple power supplies are common in server systems, in order to provide redundancy. In the largest single-chassis systems, 4 power supplies per system are common.
#### Disks
Disks are an important part of data center design, and typically store the working data for your workload, as well as your operating system files. Disks in this representation are not mandatory - it may be desirable to use a PXE network boot solution to boot your nodes into a live Linux environment on-demand, and then power them off again when the job queue is completed. This is possible with fast connections to a Storage Area Network (SAN). However, it is likely more typical that you have one drive in the chassis providing a permanent OS installation (and scratch space), and use the SAN for the rest of your data storage needs.
Disks have several important characteristics that may be relevant to simulation: storage capacity, the interface (i.e. SAS, SATA or even NVMe), the media type (whether a disk uses fast flash storage or slower spinning disks), as well as the form factor. The form factor can dictate what can be physically fit in a single chassis.
#### CPU(s)
The literal core of computing, details about our CPUs are critical to the simulation of workloads. Factors such as the core count, base and boost clocks, as well as whether Hyperthreading/SMT are supported are important to take into account. Then, there are some secondary considerations we must make - the type of CPU (SKU and brand), as well as the socket dictate whether a given CPU is even compatible with a given motherboard. The Thermal Design Power is also an accurate indication of both how much energy the CPU will consume, and how much heat it will produce. This is important for data center designers in terms of power and cooling budgets.
#### Memory
Working memory is another piece of hardware that is critical for accurate simulation of workloads. Some workloads may have higher memory complexity requirements than others, with some particularly memory intensive workloads perhaps making small performance gains with high-frequency memory. Additionally, servers typically use Error Correcting Code (ECC) memory sticks, which is worth recording when representing prefabs. Many OEM servers will not POST with unsupported DIMMs, so it may be important to represent whether ECC is being used to the user.
#### GPUs
For some GPU-bound tasks (graphics processing, image recognition, video rendering etc), a node with one or more GPUs may be required. There are quite a few GPU characteristics that may impact performance, chief among these being the amount of VRAM per card, as well as the number of GPU cores. Certain technologies (such as NVIDIA's CUDA, or AMD's openCL) may also be important to mention in the GPU configuration, as these may accelerate certain workloads.
The PCI-express generation may have a marginal performance impact, depending on the PCI bandwidth required for the compute task. The TDP is again important to consider when considering cooling and power budget. Lastly, the physical size of the card is important, especially in terms of how many PCI-e slots it requires. If the number of GPUs that can fit in a chassis is constrained by that chassis' dimensions, this is important to represent.

## Evaluation
Throughout writing this, I have determined that there are some areas on which I can improve.
* PSUs can be generic for switches too, as many enterprise-grade switches support (redundant) swappable power supplies just like servers do.
* Some chassis support multiple motherboards, allowing for seperate physical machines within a chassis that share a power supply. However, it is likely not within the scope of the thesis to exhaustively represent every possible hardware combination, so this may not be something that is implemented.
* UPSes & PDUs are also at least a little out of scope with regards to prefabs - they may serve as interesting placeholders in data center blueprints, but in representing the performance topology they are not so relevant. 
I also want to add that prefabs don't have to be exhaustive - users will be able to design their own prefabs, allowing them to represent specific hardware for their needs.