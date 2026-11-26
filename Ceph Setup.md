## Prerequisite (in all servers)

1. Install lvm2, container engine (podman or docker), ntp, ssh-server
   ```bash
   sudo apt update && sudo apt install -y lvm2 podman ntp openssh-server
   ```
2. Setup time syncronization with ntp (this can be dne with other software too)
   ```bash
   ntpq -p # for checking if ntp is working all right
   ```
3. populate /etc/hosts with server hostnames and ip
   ```
   # /etc/hosts.example
   127.0.0.1	localhost
   127.0.1.1	hostname_of_the server

   192.168.10.11   node-01
   192.168.10.12   node-02
   192.168.10.13   node-03
   192.168.10.14   node-04
   ```
4. generate ssh keys for ceph user
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
   ```
5. Copy the ssh-key to other servers
   Either use ssh-copy-id
   ```bash
   # For this to work, ssh password authentication must be enabled
   # replace $n with number of server
   # change must be made to the code according to setup convenctions
   # DO NOT BLINDLY COPY-PASTE
   for i in {1..$n}; do
       ssh-copy-id -i ~/.ssh/id_ed25519 "ceph@192.168.10.1$i"
    done
   ```
   Or just ssh into the server and append contents of the public key file `~/.ssh/id_ed25519.pub` to `~/.ssh/authorized_keys` in all other servers
6. Remove password authentication for ssh (optional but better)
7. setup ceph user to have passwordless
   ```bash
   sudo visudo -f /etc/sudoers.d/ceph
   # This one is command is not recommended,
   # just for showing what the contents of the file should be
   echo "ceph ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ceph
   ```

## Setting up node-01 (or whichever one is main)

1. Create another admin user with sudo privileges and make it ready for ssh from the management machine
   ```bash
   sudo useradd -s /bin/bash -m admin -G sudo
   ```
2. Copy the cluster ssh-key to /etc/ceph
   ```bash
   sudo cp ~/.ssh/id_e25519 /etc/ceph/ceph
   sudo cp ~/.ssh/id_e25519.pub /etc/ceph/ceph.pub
   ```
3. Install cephadm and ceph-common
   ```bash
   sudo apt install cephadm
   sudo cephadm install ceph-common

   # or you can just run
   # sudo apt install cephadm ceph-common
   ```
4. In each server, there will be one disk with an OS and other disk(s) without anything, to be used for the cluster. These disks must be empty.
   ```bash
   # for each disk to be used for the cluster
   sudo wipefs -a /dev/sdb
   sudo sgdisk --zap-all /dev/sdb
   ```
5. Bootstrap the ceph cluster
   ```bash
   # cephadm bootstrap --mon-ip <cluster-main-ip> --ssh-user <ceph-user> --ssh-private-key <private-key> --ssh-public-key <public-key>
   cephadm bootstrap --mon-ip 192.168.10.11 \
       --ssh-user ceph \
       --ssh-private-key /etc/ceph/ceph \
       --ssh-public-key /etc/ceph/ceph.pub \
       --cluster-nnetwork 192.168.10.0/24
   ```
   In the the bootstrap log, after completion, a username, password and link to the ceph cluster dashboard will be given. Use the dashboard if needed.
   ```bash
   # Post bootstrap checks
   ## check the ssh config used by ceph
   sudo cephadm shell -- ceph cephadm get-ssh-config
   # The public key should be here
   sudo ls -l /etc/ceph/
   ```
6. Add other hosts to the cluster

   ```bash
   # ceph orch host add <hostname> <host-ip>
   ceph orch host add node-02 192.168.10.12

   # check with
   ceph orch host ls
   ```
7. Add Monitors
   The bootstrapped server will have a monitor (the host with `_admin` label)
   ```bash
   # ceph orch daemon add mon <host1:ip-or-network1>
   ceph orch daemon add mon node-02:192.168.10.12
   ```
   Verify with
   ```bash
   ceph orch ps --daemon_type mon

   # cehck quorum status
   ceph quorum_status
   ```
8. Add osds (disks)
   ```bash
   # list all available storage devices and status
   ceph orch device ls

   # Uses all storage device listed as available
   ceph orch apply osd --all-available-devices

   # If any disks were unavailable during above command
   # or if you want to manually add disk in a specific host
   # ceph orch daemon add osd <host>:<device-path>
   ceph orch daemon add osd node-01:/dev/sdb

   # Verify
   ceph orch ps --daemon_type osd
   ceph osd tree
   ```
9. Configure replication
    Since this is a two node system, having replication of max 2 is practical
    Current default pool config can be checked with
    ```bash
    ceph config get global osd_pool_default_size # should be 3
    ceph config get global osd_pool_default_min_size
    ```
    To change it
    ```bash
    ceph config set global osd_pool_default_size 2
    ceph config set global osd_pool_default_min_size 2
    ```
10. Creating cephFS
    This is what will be mounted and used by us. A cephFS requires
    - data pool
    - metadata pool
    - MDS (metadata server) : this is a service
    ```bash
    ceph fs volume create cephfs

    # This command will create the metadata and data pool and spin up the MDS
    ```

    # Verify with
    ```bash
    # List the created volumes 
    ceph fs ls
    
    # check MDS status
    ceph orch ps --daemon_type mds

    # inspect the pools
    ceph osd pool ls
    ```

    In 2-node architecture, pool max size must be 2 and min size must be 1
    ```bash
    ceph osd pool get <pool-id> size
    ceph osd pool get <pool-id> min_size
    ```
    In case of conflicts, change size to 2 and min_size to 1 with
    ```bash
    ceph osd pool set <pool-id> size 2
    ceph osd pool set <pool-id> min_size 1
    ```

    Do this for data and metadata pools
