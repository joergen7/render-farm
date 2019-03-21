# render-farm
###### render farm using POV-Ray

This Cuneiform script uses the rendering language [POV-Ray](http://www.povray.org/) to render a 3D scene. To distribute the computation the scene is divided in tiles and each POV-Ray instance computes one such tile. Next, the script montages the tiles horizontally to lines. Lastly, the script montages the lines vertically to form the complete scene.

The purpose of this script is to demonstrate that it is possible to speed up the rendering of a 3D scene using Cuneiform to manage a cluster of Raspberry Pis.

## Test Setup

### Simple Setup

Install the following packages:
- `erlang`
- `imagemagick`
- `racket`
- `povray`


### Local Cuneiform Setup

Download [Cuneiform 3.0.4](https://cuneiform-lang.org/download/cuneiform-3.0.4-OTP-19.0.zip), unzip the content and place the binaries under `~/bin`.

### Run Locally

Install the render farm script and POV-RAY scene by entering

    git clone https://github.com/joergen7/render-farm.zip`

In the `render-farm` directory and enter

    cuneiform render-farm.cfl

## Distributed Setup

Let's say we have picked one machine to add as the CRE server called `server`. All other machines act as Cuneiform workers called `workerX` where `X` is a unique id. One last machine acts as the Cuneiform client called `client`. The user on each machine is called `jorgen`.

Every machine needs the packages
- `erlang`
- `nfs-common`

Only the server:
- `nfs-kernel-server`

Only the workers:
- `imagemagick`
- `racket`
- `povray`

### NFS setup

To run a script distributedly, Cuneiform needs a posix-compliant distributed file system. Since the data we exchange is fairly small, an [NFS](https://wiki.ubuntuusers.de/NFS/) is enough.


Install `nfs-common` on all nodes. Only on the NFS server install `nfs-kernel-server` and enable the NFS server by entering

    sudo systemctl enable nfs-kernel-server.service

You can use `rpcinfo -p` to find out if NFS is indeed running.

Create the following directories on the NFS server

    mkdir -p -m 0777 /exports/bin
    mkdir -p -m 0777 /exports/data
    mkdir -p -m 0777 /exports/repo

Edit the file `/etc/exports` on the NFS server

    /exports/bin  *(rw,sync,subtree_check,all_squash)
    /exports/data *(rw,sync,subtree_check,all_squash)
    /exports/repo *(rw,sync,subtree_check,all_squash)

Edit the file `/etc/fstab` on all workers:

    server:/exports/bin  /home/jorgen/bin   nfs  ro         0  0
    server:/exports/data /home/jorgen/data  nfs  ro,noexec  0  0
    server:/exports/repo /home/jorgen/repo  nfs  rw,noexec  0  0

Create directories for each mount point.

Note that the default setup for users automatically includes `~/bin` into the `PATH` variable. So no further Configuration is necessary.

On the server edit the crontab by entering `crontab -e`

    @reboot    rm -rf /exports/repo/*

This makes the server delete its repo directory at reboot.

### Cuneiform Setup

#### Installation

On the server download the current Cuneiform distribution by entering

    wget https://cuneiform-lang.org/download/cuneiform-3.0.4-OTP-19.0.zip

and extract the content to /exports/bin.

### Configuration

On all nodes create a file `.config/cuneiform/cf_worker.json` for in the home directory of the user `jorgen` and enter

    {
      "n_wrk":    4,
      "cre_node": "cre@server",
      "wrk_dir":  "/tmp/_cuneiform/wrk",
      "repo_dir": "/home/jorgen/repo",
      "data_dir": "/home/jorgen/data"
    }

On the client node create a file `.config/cuneiform/cf_client.json` and enter

    {
      "cre_node": "cre@server"
    }

### Automatic start up

On the server create a crontab entry by entering `crontab -e` as user `jorgen`

    * * * * *    /home/jorgen/bin/cre > /tmp/cre.log 2>&1

On the remaining nodes enter

    * * * * *    sleep 30; /home/jorgen/bin/cf_worker > /tmp/cf_worker.log 2>&1

The file `.erlang.cookie` in the user's home directory must be the same on all computers.

### Placing the Input Data

Put the file `scene.pov` in `exports/data` on the `server` machine.

### Run Distributed

Booting all machines should
- start an NFS server on `server`
- connect all `workerX` to the NFS server on `server`
- delete the `repo` directory on `server`
- start a Cuneiform CRE on `server`
- start a Cuneiform worker on each `workerX`

Now on the client machine enter

    cf_client -c cre@server render-farm.cfl