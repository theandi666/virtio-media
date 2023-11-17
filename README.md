# Virtio-media

This is a virtio protocol definition and companion Linux guest kernel driver
for virtualizing media devices using virtio, following the same model (and
structures) as V4L2. It can be used to virtualize cameras, codec devices, or
any other device supported by V4L2.

V4L2 is a UAPI that allows a less privileged entity (user-space) to use video
hardware exposed by a more privileged entity (the kernel). Virtio-media is an
encapsulation of this API into virtio, turning it into a virtualization API for
all classes of video devices supported by V4L2, where the host plays the role
of the kernel and the guest the role of user-space.

The host is therefore responsible for presenting a virtual device that behaves
like an actual V4L2 device, which the guest can control.

This repository includes a simple guest Linux kernel module supporting this
protocol. On the host side, devices can be implemented by several ways:

1. By forwarding a V4L2 device from the host into a guest. This works if the
   device is already supported by V4L2 on the host.
2. By emulating a V4L2 device on the host, from the actual interface that the
   device provides.

Note that virtio-media does not require the use of a V4L2 device driver or of
Linux on the host or guest side - V4L2 is only used as a host-guest protocol,
and both sides are free to convert it from/to any model that they wish to use.

The complete definition of V4L2 structures and ioctls can be found under the
[V4L2 UAPI
documentation](https://www.kernel.org/doc/html/latest/userspace-api/media/index.html),
which should be used as a companion to this document.

## Driver status

The driver should be working and supporting most V4L2 features, with the
exception of the following:

* [Read/Write API](https://www.kernel.org/doc/html/v4.8/media/uapi/v4l/rw.html), which is obsolete and inefficient.
* Overlay interface and associated ioctls, i.e.
    * `VIDIOC_OVERLAY`
    * `VIDIOC_G/S_FBUF`
* `DMABUF` buffers (this will be supported at least for virtio objects)
* `VIDIOC_EXPBUF` (to be implemented)
* `VIDIOC_G/S_EDID` (to be implemented if it makes sense in a virtual context)
* Media API and requests. This will probably be supported in the future behind
  a feature flag.

## Virtqueues

There are two queues in use:

0
: commandq - queue for driver commands and device responses to these commands.

1
: eventq - queue for events sent by the device to the driver.

## Configuration area

The configuration area contains the following information:

```
struct virtio_v4l2_config {
    /// The device_caps field of struct video_device.
    u32 device_caps;
    /// The vfl_devnode_type of the device.
    u32 device_type;
    /// The `card` field of v4l2_capability.
    u8 card[32];
}
```

## Protocol

All structures managing the virtio protocol are defined and documented in
`protocol.h`. Please refer to this file whenever a `virtio_media_cmd_*` or
`virtio_media_resp_*` structure is mentioned.

Commands are queued on the `commandq` by the driver for the device to process.
They all start by an instance of `struct virtio_media_cmd_header` and include
device-writable descriptors for the device to write the result of the command
in a `struct virtio_media_resp_header`.

The errors returned by each command are standard Linux kernel error codes. For
instance, a command that contains invalid options will return `EINVAL`.

Events are sent on the `eventq` by the device for the driver to handle. They
all start by an instance of `struct virtio_media_event_header`.

## Session management

In order to use the device, the driver needs to open a session. This act is
equivalent to opening the `/dev/videoX` device file of the V4L2 device.
Depending on the type of device, it may be possible to open several sessions
concurrently.

A session is opened by queueing a `struct virtio_media_cmd_open` along with a
descriptor to receive a `struct virtio_media_resp_open` to the commandq. An
open session can be closed with `struct virtio_media_cmd_close`.

While the session is opened, its ID can be used to perform actions on it, most
commonly V4L2 ioctls.

## Ioctls

Ioctls are the main way to interact with V4L2 devices, and therefore
virtio-media features a command to perform an ioctl on an open session.

In order to perform an ioctl, the driver queues a `struct
virtio_media_cmd_ioctl` along with a descriptor to receive a `struct
virtio_media_resp_ioctl` on the commandq. The code of the ioctl can be
extracted from the
[videodev2.h](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/videodev.html)
header file, which defines the ioctls' codes, type of payload, and direction.
For instance, the `VIDIOC_G_FMT` ioctl is defined as follows:

```
#define VIDIOC_G_FMT            _IOWR('V',  4, struct v4l2_format)
```

This tells us that its ioctl code is `4`, that its payload is a `struct
v4l2_format`, and that its direction is `WR`, i.e. the payload is written by
both the driver and the device.

### Ioctls payload

The payload of an ioctl in the descriptor chain follows the command structure,
the reponse structure, or both depending on the direction:

* An `_IOR` ioctl is read-only for the driver, meaning the payload follows the
	response in the device-writable section of the descriptor chain.
* An `_IOW` ioctl is read-only for the device, meaning the payload follows the
	command in the driver-writable section of the descriptor chain.
* An `_IORW` ioctl is writable by both the device and driver, meaning the
	payload must follow both the command in the driver-writable section of the
	descriptor chain, and the response in the device-writable section.

For instance, the `VIDIOC_G_STD` ioctl is defined as follows:

```
#define VIDIOC_G_STD		 _IOR('V', 23, v4l2_std_id)
```

Thus, its layout on the commandq will be:

```
+-------------------------------------+
| struct virtio_media_cmd_ioctl       |
+=====================================+
| struct virtio_media_resp_ioctl      |
+-------------------------------------+
| v4l2_std_id                         |
+-------------------------------------+
```

(in these diagrams, the `====` line signals the delimitation between
device-readable and device-writable descriptors).

`VIDIOC_SUBSCRIBE_EVENT` is defined as follows:

```
#define	VIDIOC_SUBSCRIBE_EVENT	 _IOW('V', 90, struct v4l2_event_subscription)
```

Meaning its layout on the commandq will be:

```
+-------------------------------------+
| struct virtio_media_cmd_ioctl       |
+-------------------------------------+
| struct v4l2_event_subscription      |
+=====================================+
| struct virtio_media_resp_ioctl      |
+-------------------------------------+
```

Finally, `VIDIOC_G_FMT` is a `WR` ioctl:

```
#define VIDIOC_G_FMT            _IOWR('V',  4, struct v4l2_format)
```

Its layout on the commandq will thus be:

```
+-------------------------------------+
| struct virtio_media_cmd_ioctl       |
+-------------------------------------+
| struct v4l2_format                  |
+=====================================+
| struct virtio_media_resp_ioctl      |
+-------------------------------------+
| struct v4l2_format                  |
+-------------------------------------+
```

A common optimization for `WR` ioctls is to provide the payload using
descriptors that both point to the same buffer. This mimics the behavior of
V4L2 ioctls where the data is only passed once and used as both input and
output by the kernel.

In case of success, the device MUST always write the payload in the
device-writable part of the descriptor chain.

In case of failure, the device is free to write the payload in the
device-writable part of the descriptor chain or not. Some errors may still
result in the payload being updated, and in this case the device is expected to
write the updated payload (for instance, `G_EXT_CTRLS` may return `EINVAL` but
update the `size` member of the requested controls if the provided size was not
enough). If the device has not written the payload after an error, the driver
MUST assume that the payload has not been modified.

### Handling of pointers in ioctl payload

A few structures used as ioctl payloads contain pointers the link to further
data needed for the ioctl. There are notably:

* The `planes` pointer of `struct v4l2_buffer`, which size is determined by the
	`length` member,
* The `controls` pointer of `struct v4l2_ext_controls`, which size is
	determined by the `count` member.

If the size of the pointed area is determined to be non-zero, then the main
payload is immediately followed by the pointed data in their order of
appearance in the structure, and the pointer value itself is ignored by the
device, which must also return the value initially passed by the driver. For
instance, for a `struct v4l2_ext_controls` which `count` is `16`:

```
+-------------------------------------+
| struct v4l2_ext_controls            |
+-------------------------------------+
| array of 16 struct v4l2_ext_control |
+-------------------------------------+
```

Individual controls may also have their own `ptr` member set if their `size` is
non-zero. In that case that payload will follow the array of
`v4l2_ext_control`, in their order of appearance. For instance, if controls `3`
and `7` have their `size` member set to `256` and `1024` respectively, then the
driver will lay the payload out as follows:

```
+-------------------------------------+
| struct v4l2_ext_controls            |
+-------------------------------------+
| array of 16 struct v4l2_ext_control |
+-------------------------------------+
| 256 bytes payload of control 3      |
+-------------------------------------+
| 1024 bytes payload of control 7     |
+-------------------------------------+
```

The device will be able to reassign this data properly by looking at the `size`
member of each retrieved control, and processing the corresponding amount of
data from the descriptor chain.

Important note: pointer data does not need to be copied into new buffers to be
inserted in the descriptor chain. Instead the driver can use chained
descriptors that point to where the data originally stands. This is especially
important for `USERPTR` buffers, since the pointed data can be quite large.
Using chained descriptors as a ad-hoc scatter/gather list prevents costly
guest/host (and back again) copies. 

A corrolary of this note is that care should be taken to keep the commandq
large enough to accomodate long descriptor chains, as a single `USERPTR` buffer
can in the worst case require hundreds of descriptors to be queued.

This side-effect is mitigated by mapping pointers marked as `__user` in V4L2
only once, in the command section of the descriptor chain, for `WR` ioctl.
Since the device will write the data in the addresses provided, mentioning
these twice would be redundant anyway.

### Unsupported ioctls

A few ioctls are replaced by other, more suitable mechanisms. If being
requested these ioctls, the device must return the same response as it would
for an unknown ioctl, i.e. `ENOTTY`.

* `VIDIOC_QUERYCAP` is replaced by reading the configuration area.
* `VIDIOC_DQBUF` is replaced by a dedicated event.
* `VIDIOC_DQEVENT` is replaced by a dedicated event.
* `VIDIOC_G_CTRL` and `VIDIOC_S_CTRL` are implemented using
  `VIDIOC_G_EXT_CTRLS` and `VIDIOC_S_EXT_CTRLS` respectively.
* `VIDIOC_G_JPEGCOMP` and `VIDIOC_S_JPEGCOMP` are deprecated and replaced by
  the controls of the JPEG class.
* `VIDIOC_LOG_STATUS` is a guest-only operation and shall not be implemented by
  the host.

## Events

Events are a way for the device to inform the driver about asynchronous events
that it should know about. In virtio-media, they are used as a replacement for
the `VIDIOC_DQBUF` and `VIDIOC_DQEVENT` ioctls and the polling mechanism, which
would be impractical to implement on top of virtio.

### Dequeued buffer events

A `struct virtio_media_event_dqbuf` event is queued on the eventq by the device
every time a buffer previously queued using the `VIDIOC_QBUF` ioctl is done
being processed and can be used by the driver again. This is like an implicit
`VIDIOC_DQBUF` ioctl.

Pointer values in the `struct v4l2_buffer` and `struct v4l2_plane` are
meaningless and must be ignored by the driver. It is recommended that the
device sets them to `NULL` in order to avoid leaking potential host addresses.

Note that in the case of a `USERPTR` buffer, the `struct v4l2_buffer` used as
event payload is not followed by the buffer memory: since that memory is the
same that the driver submitted with the `VIDIOC_QBUF`, it would be redundant to
have it here.

### Dequeued V4L2 event event

A `struct virtio_media_event_event` event is queued on the eventq by the device
every time an event the driver previously subscribed to using the
`VIDIOC_SUBSCRIBE_EVENT` ioctl has been signaled. This is like an implicit
`VIDIOC_DQEVENT` ioctl.

## Memory types

The semantics of the three V4L2 memory types (`MMAP`, `USERPTR` and `DMABUF`)
can easily be mapped to a guest/host context.

### MMAP

In virtio-media, `MMAP` buffers are provisioned by the host, just
like they are by the kernel in regular V4L2. Similarly to how userspace can map
a `MMAP` buffer into its address space using `mmap` and `munmap`, the
virtio-media driver can map host buffers into the guest space by queueing the
`struct virtio_media_cmd_mmap` and `struct virtio_media_cmd_munmap` commands to
the commandq.

### USERPTR

In virtio-media, `USERPTR` buffers and provisioned by the guest,
just like they are by userspace in regular V4L2. Instances of `struct
v4l2_buffer` and `struct v4l2_plane` of this type are followed by a series of
descriptors mapping the buffer backing memory in guest space.

For the host convenience, the backing memory must start with a new descriptor -
this allows the host to easily map the buffer memory to render into it instead
of having to do a copy.

The host must not alter the pointer values provided by the guest, i.e. the
`m.userptr` member of `struct v4l2_buffer` and `struct v4l2_plane` must be
returned to the guest with the same value as it was provided.

### DMABUF

In virtio-media, `DMABUF` buffers are provisioned by a virtio object, just like
they are by a DMABUF in regular V4L2.

