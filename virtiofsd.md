# virtiofsd

Recently I came across virtiofsd and thought I'd try it out on my local QEMU
usage setup. It worked surprisingly well, and its great to have a virtualized
device written in rust, with the possibility to run in sandboxed mode by default.

First, I run virtiofsd as follows:
```
# ./target/release/virtiofsd \
    --log-level=debug \
    --socket-path=/tmp/virtiofs \
    --shared-dir=/tmp/shared_dir \
    --cache=never --sandbox=namespace --seccomp=kill
```

Then start QEMU with the following additional arguments:
```
        -chardev socket,id=char0,path=/tmp/virtiofs \
        -device vhost-user-fs-pci,chardev=char0,tag=myfs \
    	-object memory-backend-memfd,id=mem,size=8G,share=on \
    	-numa node,memdev=mem \
```

Once the guest VM starts up, we can see debug log messages along the lines of:
```
[2024-07-08T03:39:41Z INFO  virtiofsd] Waiting for vhost-user socket connection...
[2024-07-08T03:39:56Z INFO  virtiofsd] Client connected, servicing requests
[2024-07-08T03:40:10Z DEBUG virtiofsd] HIPRIO_QUEUE_EVENT
[2024-07-08T03:40:10Z DEBUG virtiofsd] QUEUE_EVENT
[2024-07-08T03:42:34Z DEBUG virtiofsd] QUEUE_EVENT
```

Within the guest, do a mount, and we are good to go.
```
$ sudo mount -t virtiofs myfs /mnt
```

Debug log messages upon doing the mount are:
```
[2024-07-08T03:42:34Z DEBUG virtiofsd::server] Received request: opcode=Init (26), inode=0, unique=2, pid=0
[2024-07-08T03:42:34Z DEBUG virtiofsd::server] Replying OK, header: OutHeader { len: 80, error: 0, unique: 2 }
```

virtiofsd quits as soon as the VM is powered down, though there likely are ways around
this-- the daemon is built with the intention of being shared across multiple VMs.

There is an [excellent talk](https://www.youtube.com/watch?v=EIVOzTsGMMI&t=1074s) by
QEMU maintainer Stefan on how virtiofsd works, I encourage you to go watch that talk.

Summarizing, this is how it works:
* the transport used is VirtIO with shared memory extensions; the protocol is FUSE.
* FUSE requests(such as INIT, LOOKUP, OPEN, READ etc) are sent for file operations;
and responses are sent by virtiofsd which is a FUSE application server.
* data is still being copied from the guest RAM to host RAM; this can be avoided by
using DAX.

