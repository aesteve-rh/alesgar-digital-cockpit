---
title: "Introducing vhost-device-video"
date: 2023-11-15T17:53:23+01:00
draft: false
---

Recently, as part of the automotive efforts at Red Hat, we have developed
and/or collaborated in various
[rust-vmm/vhost-device](https://github.com/rust-vmm/vhost-device) vhost
backends, e.g., [virtIO SCMI](https://github.com/rust-vmm/vhost-device/tree/main/vhost-device-scmi)
with Milan Zamazal,
[virtIO sound](https://github.com/rust-vmm/vhost-device/tree/main/staging/vhost-device-sound)
with Dorinda Bassey and Matias Ezequiel among others,
or the one I was more involved personally, [virtIO video](https://github.com/rust-vmm/vhost-device/tree/main/staging/vhost-device-video).
VirtIO video is, as you may infer from its name, a virtualization protocol for
codec devices. This post comes soon after the `vhost-user-video` crate has
been upstreamed, and hopefully will be the start of a series of posts
discussing various aspects of the crate, at various levels of detail and
technicality.

Here, I will discuss some general considerations, and then explain how can
you run this device and test it yourself. This post assumes that you are
familiar with virtIO architecture and nomenclature, and the importance of
video data in the recent years. There are others blogs that already discussed
this better than I could. So let’s start already and focus on getting the
crate running, shall we?

First of all, you may have realised that there is no video in the latest
[virtIO specification](https://docs.oasis-open.org/virtio/virtio/v1.2/csd01/virtio-v1.2-csd01.html) release, so what about about having an upstream implementation?
And while it’s true that specification has not been officialy released,
there has also been a lot of interest lately, with some discussions on the
future of the specification, and a good bunch of RFC patches to date
(latest patch being [v8](https://www.spinics.net/lists/linux-media/msg234972.html)).

So after our initial investigation phase, and taking over a Qemu
[patch](https://www.mail-archive.com/qemu-devel@nongnu.org/msg950676.html) from
Linaro, we decided to put efforts into a rust-vmm implementation.
The rust-vmm maintaners, specially Stefano Garzarella, were very open and
helpful from the beginning. A specific `staging` folder was created for WiP
implementations, or, as in this case, those for which specification is still
under discussion. So after all of this was agreed on, was just a matter of time
we got it upstreamed and available for the community.

![Video meme](/img/video-meme.jpg)

Nonetheless, we opted to work with version 3 of the specification [patch](https://drive.google.com/file/d/1jOsS2WdVhL4PpcWLO8Zukq5J0fXDiWn-/view), which is already available in some
out-of-tree kernels, as [ChromeOS](https://github.com/OpenSynergy/android-kernel-common/tree/opsy/android11-5.4-trout/drivers/media/virtio)
and [Android](https://github.com/OpenSynergy/android-kernel-common/tree/opsy/android11-5.4-trout/drivers/media/virtio), and even `crosVM` also have its own
[experimental](https://crosvm.dev/book/devices/video.html) implementation of
the backend. This allowed us to test it early. And, since our
use case at Red Hat revolves around Android, it only made sense to keep the
specification compatible. However, as the specification matures, we do not
discard updating the implementation if that would help in getting it published.

## Running a virtio-video example

"Great!", you may think, "but how can I use it?". Well, good question...

### Prerequisites

Firstly, virtIO video is used to expose a codec device to a running guest.
Thus, you will need such a device on your machine to run the backend.
But worry not, if you do not have one, there are a number of virtual codec
implementations that can be used. These drivers mimic a real device and handle
the system calls and perform pure software decode/encode operations.
In my case, I have used [vicodec](https://lwn.net/Articles/760650/) to this end.
Recent Fedora kernels do have the module available, so you do not even need to
compile it yourself. You can easily enable it by doing:

```shell
host#  sudo modprobe vicodec multiplanar=1
host#  v4l2-ctl -d3 --info
Driver Info:
        Driver name      : vicodec
        Card type        : vicodec
        Bus info         : platform:vicodec
        Driver version   : 6.5.6
        Capabilities     : 0x84204000
                Video Memory-to-Memory Multiplanar
                Streaming
                Extended Pix Format
                Device Capabilities
        Device Caps      : 0x04204000
                Video Memory-to-Memory Multiplanar
                Streaming
                Extended Pix Format
Media Driver Info:
        Driver name      : vicodec
        Model            : vicodec
        Serial           : 
        Bus info         : platform:vicodec
        Media version    : 6.5.6
        Hardware revision: 0x00000000 (0)
        Driver version   : 6.5.6
Interface Info:
        ID               : 0x0300001a
        Type             : V4L Video
Entity Info:
        ID               : 0x0000000f (15)
        Name             : stateful-decoder-source
        Function         : V4L2 I/O
        Pad 0x01000010   : 0: Source
          Link 0x02000016: to remote pad 0x1000012 of entity 'stateful-decoder-proc' (Video Decoder): Data, Enabled, Immutable
```

Secondly, you must consider that at the moment of writing this post, only
a [v4l2](https://es.wikipedia.org/wiki/Video4Linux) decoder backend is available.
Patches adding more backends for different platforms and encoders are welcome,
but in the meantime, you will need to work in a Linux machine.

### Setting up the guest

Unless you use one of the aforementioned systems that already have a
virtio driver for video devices, you will need to compile a kernel for
your guest. You have a kernel available at: https://github.com/aesteve-rh/linux.

Before compiling the kernel, make sure you enable the right modules:

    CONFIG_MEDIA_SUPPORT=y
    CONFIG_MEDIA_TEST_SUPPORT=y
    CONFIG_V4L_TEST_DRIVERS=y
    CONFIG_VIRTIO_VIDEO=y
    CONFIG_GDB_SCRIPTS=y
    CONFIG_DRM_VIRTIO_GPU=y

In this example we will use `Qemu` as our VMM. However, `Qemu` does not have
a video PCI device available to enable on the command line. We need one to
make sure that the driver we just compiled finds a compatible device exposed.
There is a downstream patch that you can use to be able to do exactly that:
[Qemu virtio-video-v3](https://github.com/qemu/qemu/compare/master...aesteve-rh:qemu:virtio-video-v3).
So yeah, you guessed it, you also need to [build](https://github.com/qemu/qemu#building)
`Qemu` to make it work.

### How to test it

The daemon should be started first (video3 typically is the stateful video,
as shown above, but it may change on your machine):

```shell
host# vhost-user-video --socket-path=/tmp/video.sock --v4l2-device=/dev/video3 --backend=v4l2-decoder
```

The QEMU invocation needs to create a chardev socket the device can use to communicate as well as share the guests memory over a memfd.

```shell
host# qemu-system								                            \
    -device vhost-user-video-pci,chardev=video,id=video                     \
    -chardev socket,path=/tmp/video.sock,id=video                           \
    -m 4096 		        					                            \
    -object memory-backend-file,id=mem,size=4G,mem-path=/dev/shm,share=on	\
    -numa node,memdev=mem							                        \
    ...
```

After booting, the device should be available at `/dev/video0`:

```shell
guest# v4l2-ctl -d/dev/video0 --info
Driver Info:
        Driver name      : virtio-video
        Card type        : 
        Bus info         : virtio:stateful-decoder
        Driver version   : 6.1.0
        Capabilities     : 0x84204000
                Video Memory-to-Memory Multiplanar
                Streaming
                Extended Pix Format
                Device Capabilities
        Device Caps      : 0x04204000
                Video Memory-to-Memory Multiplanar
                Streaming
                Extended Pix Format
```
Yay! The device is picked up by the virtio driver, and is recognized as a
`stateful-decoder`. Now you can run some example v4l2-ctl decode command:

```shell
guest# v4l2-ctl -d0 -x width=640,height=480 -v width=640,height=480,pixelformat=YU12 \
    --stream-mmap --stream-out-mmap --stream-from test_640_480-420P.fwht             \
    --stream-to out-test-640-480.YU12
```

As you can see, the video format in the example is FWHT
(Fast Walsh Hadamard Transform) format, which is the format supported
by `vicodec`, but you can use another format in your tests.

Happy testing! And I encourage you to report any issues you may find.

## Future plans

The future of the device is tied to the future of the virtio specification.
Us, the community, have the power to drive this to a mature enough state that
makes the cut for the next virtio specification release.

In the meantime, patches for tha backend are welcome! Working on an older
version of the specs may feel counter-productive, but improving the backend
will help updating it to a newer specification later on. Implementing a
robust backend solution now will definitively have an impact on how the final
version looks like, and how it deals with the challenges of the specification.

And of course, stay tuned for more content and updates on the state of
virtIO video!