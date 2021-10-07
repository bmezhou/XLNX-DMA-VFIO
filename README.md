[[_TOC_]]


# Xilinx DMA with VFIO introduction

The **feasibility** of Xilinx DMA subsystem for PCIe working with VFIO (user space driver) is investigated in this repository.

* VFIO: Virtual Function I/O (https://www.kernel.org/doc/html/latest/driver-api/vfio.html)
* Xilinx DMA: Xilinx DMA/bridge subsystem for PCI Express (PCIe) implements a high performance, configurable scatter gather DMA for use with the PCI Express 2.1 and 3.x integrated block (https://www.xilinx.com/support/documentation/ip_documentation/xdma/v4_1/pg195-pcie-dma.pdf)

The repository is inspired by NVIDIA jetson-rdma-picoevb 
(https://github.com/NVIDIA/jetson-rdma-picoevb)

# Some Marcos and auxiliary functions

## min_t and max_t

```C
#define min_t(type, x, y) \
  ({ type __x = (x); \
     type __y = (y); \
     __x < __y ? __x: __y; })

#define max_t(type, x, y) \
  ({ type __x = (x); \
     type __y = (y); \
     __x > __y ? __x: __y; })
     
```

## BIT()

BIT is a macro defined in ```include/linux/bitops.h```

```C
#define BIT(nr) (1UL << (nr))
```


## Xilinx PCI-express registers address for read and write

```C
#define XLNX_REG(_target_, _channel_, _reg_) \
	((XLNX_DMA_TARGET_##_target_ << 12) | \
	((_channel_) << 8) | \
	XLNX_DMA_##_reg_)
```

Note that symbol '\#\#' in C language marco is to 'connect with the left string'.

Take ```XLNX_REG(C2H, 0, H2C_CTRL)``` as an example. The expression can be expended to ```(1 << 12) | (0 << 8) | (0x04)``` (according to the following macros)

```C
// in picoevb-rdma.h
#define XLNX_DMA_TARGET_C2H			1
#define XLNX_DMA_H2C_CTRL			0x04

// XLNX_REG(C2H, 0, H2C_CTRL) ==
// (( XLNX_DMA_TARGET_C2H << 12 ) | \
// (( 0 ) << 8) | \
// XLNX_DMA_H2C_CTRL )
```

# RDMA control (Xilinx PG195)

The basic steps of Xilinx DMA in kernel mode driver are composed of

1. Pin user-space memory
2. Obtain page table 
3. Map DMA addresses and buffers
4. Structure descriptor chain list
5. Start DMA operation
6. Unmap the DMA buffers

In VFIO module, steps 1-3 are performed implicitly by ```ioctl()```. Steps 4-5 have to be done explicitly by user-space program. Step 6 is left to be tested. (See Section **Data coherence**)

## Descriptors (or structure fields)

According to Xilinx document PG-195, for example, the descriptor format is defined as

```C
struct xlnx_dma_desc {
	u32 control;    // Control value
	u32 len;        // Length of the data in bytes
	u32 src_adr;    // Source address for H2C and 
	u32 src_adr_hi; //   memory mapped transfers
	u32 dst_adr;    // Destination address for C2H and 
	u32 dst_adr_hi; //   memory mapped transfers
	u32 nxt_adr;    // Address of the next descriptor
	u32 nxt_adr_hi; //   (Bus address)
} __packed;
```

For example, in NVIDIA jetson-rdma-picoevb, the descriptor is initialized as

```C
// Create descriptor
desc = pevb->descs_ptr;
desc->control = XLNX_DMA_DESC_CONTROL_MAGIC |
	XLNX_DMA_DESC_CONTROL_EOP |
	XLNX_DMA_DESC_CONTROL_COMPLETED |
	XLNX_DMA_DESC_CONTROL_STOP;
desc->len        = len;
desc->src_adr    = pcie_addr & 0xffffffffU;
desc->src_adr_hi = pcie_addr >> 32;
desc->dst_adr    = ram_offset & 0xffffffffU;
desc->dst_adr_hi = ram_offset >> 32;
desc->nxt_adr    = 0;
desc->nxt_adr_hi = 0;
```

Note that the addresses in the descriptor should all be PCI-e **bus addresses** to be accessed by PCI-e devices.

With VFIO, the PCI-e bus addresses can be replaced simply by user **virtual address**. In this repository, function ```build_sgt()``` is for this purpose.

## SG-DMA initialization

The DMA operation is initialized by writing a range of registers, e.g.,

* Feed descriptor address (location)
* Enable IRQs
* Arm Performance counters
* Start DMA (see the following code)

```C
reg = XLNX_REG(H2C, 0, H2C_CTRL) + chan_offset;
val = (XLNX_DMA_H2C_CTRL_IE_DESC_ERR_MASK <<
		XLNX_DMA_H2C_CTRL_IE_DESC_ERR_SHIFT) |
	(XLNX_DMA_H2C_CTRL_IE_WRITE_ERR_MASK <<
		XLNX_DMA_H2C_CTRL_IE_WRITE_ERR_SHIFT) |
	(XLNX_DMA_H2C_CTRL_IE_READ_ERR_MASK <<
		XLNX_DMA_H2C_CTRL_IE_READ_ERR_SHIFT) |
	XLNX_DMA_H2C_CTRL_IE_IDLE_STOPPED |
	XLNX_DMA_H2C_CTRL_IE_INVALID_LEN |
	XLNX_DMA_H2C_CTRL_IE_MAGIC_STOPPED |
	XLNX_DMA_H2C_CTRL_IE_ALIGN_MISMATCH |
	XLNX_DMA_H2C_CTRL_IE_DESC_COMPLETED |
	XLNX_DMA_H2C_CTRL_IE_DESC_STOPPED |
	XLNX_DMA_H2C_CTRL_RUN;

pevb_writel(pevb, BAR_DMA, val, reg);
```
Upon finishing the DMA operations, registers should be reset and resources should be released.



## Data coherence

(a critical topic)

**Re: [vfio-users] VFIO and coherency with DMA**

https://listman.redhat.com/archives/vfio-users/2020-February/msg00005.html

> From: Alex Williamson <alex williamson redhat com> <br />
> To: "Stark, Derek" <Derek Stark molex com> <br />
> Cc: "vfio-users redhat com" <vfio-users redhat com> <br />
> Subject: Re: [vfio-users] VFIO and coherency with DMA <br />
> Date: Thu, 13 Feb 2020 09:16:48 -0700 <br />

On Thu, 13 Feb 2020 10:02:26 +0000

"Stark, Derek" <Derek Stark molex com> wrote:

> Hello,
> 
> I've been experimenting with VFIO with one of our FPGA cards using a
> Xilinx part and XDMA IP core. It's been smooth progress so far and
> I've had no issues with bar access and also DMA mapping/transfers to
> and from the card. All in all, I'm finding it a very nice userspace
> driver framework.
> 
> I'm hoping someone can help clarify my understanding of how VFIO
> works for DMA in terms of coherence. I'm on a standard x86_64 Intel
> Xeon platform.
> 
> In the  code I see: ```(/include/uapi/linux/vfio.h)```
> ```C
> /* 
> * IOMMU enforces DMA cache coherence (ex. PCIe NoSnoop stripping).  This
> * capability is subject to change as groups are added or removed.
> */ 
>
> #define VFIO_DMA_CC_IOMMU                               4
> ```
>
> Which implies that IOMMU sets the mappings up as coherent.... is this
> understanding correct?

No, this is a mechanism for reporting the cache coherency of the IOMMU.
For example, KVM uses this to understand whether it needs to emulate
wbinv instructions for cases where the DMA is not coherent.  There's
nothing vfio can specifically do in the IOMMU mapping to make a DMA
coherent afaik.

> I'm more used to having scatter gather based DMAs where you need to
> sync for the CPU or the device depending upon who owns/accesses the
> memory.
> 
> The use case I am specifically looking at is if a DMA mapping is
> setup through VFIO and then left open whilst data is transferred from
> the device to host memory and then the CPU is processing this data.
> The pinned/mapped data buffer is reused repeatedly as part of a ring
> of buffers. It's only at the point of closing down this application
> that the buffer would be unmapped in vfio.
> 
> Is there any sync type functions or equivalents I need to be aware of
> in this case? Can VFIO DMA mapped memory buffers be safely used in
> this way?

It can, but you need to test that cache coherence extension above to
know whether the processor is coherent with DMA.  If it's not then you
need to invalidate the processor cache before you pull in new data from
the device or else you might just be re-reading stale data from the
cache.  Thanks,

Alex