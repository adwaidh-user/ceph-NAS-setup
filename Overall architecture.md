2. **With 2 servers and 4 disks**
```mermaid
flowchart TB
    SW["Local Switch"]
    
    subgraph CLUSTER["Ceph NAS Cluster — 2 Servers"]
        direction LR
        
        subgraph S1["Server 1(192.168.40.176/24)"]
	        direction TB
            MON1["ceph-mon"]
            MGR1["ceph-mgr<br/>Active"]
            MDS1["ceph-mds<br/>Active"]
            D1["OS"]
            D2["osd1"]
            D3["osd2"]
            D4["osd3"]
        end
        
        subgraph S2["Server 2(192.168.10.12/24)"]
	        direction TB
            MON2["ceph-mon"]
            MGR2["ceph-mgr<br/>Standby"]
            MDS2["ceph-mds<br/>Standby"]
            D5["OS"]
            D6["osd4"]
            D7["osd5"]
            D8["osd6"]
        end
    end
    
    SW --- S1
    SW --- S2
    
    classDef active fill:#d1fae5,stroke:#059669,color:#064e3b
    classDef standby fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef disk fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef switch fill:#e5e7eb,stroke:#4b5563,color:#111827
    
    class MGR1,MDS1 active
    class MGR2,MDS2 standby
    class D1,D2,D3,D4,D5,D6,D7,D8 disk
    class SW switch
```

## Common prerequisites in hosts
- container engine (docker / podman) installed
- ntp / chronyd (enable timesynchronization)
- lvm2 installed
- install ceph-common (via cephadm install or apt)

# Ceph Quorum

> Ceph Quorum is a state in which machine failures upto a limit can be overcome by other monitor.

In an evironment with N systems, for achiving quorum, there must be ( N / 2 ) + 1 ceph-mon isntances at minimum

