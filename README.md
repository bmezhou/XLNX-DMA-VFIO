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

HOwever, in VFIO module, steps 1-3 are performed internally. Steps 4-5 have to be done in user-space program.

## Descriptors (or structure fields)

According to Xilinx document PG-195, the descriptor format is defined as

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

In NVIDIA jetson-rdma-picoevb, the descriptor is initialized as (see Tables 5 and 6 in PG195)

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

With VFIO, the PCI-e bus addresses can be replaced simply by user **virtual address**. In this repository, function build_link() performs this.

## SG-DMA initialization



## Data coherence

(an critical topic)