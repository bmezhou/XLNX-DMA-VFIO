[[_TOC_]]


# Xilinx DMA with VFIO introduction

The **feasibility** of Xilinx DMA subsystem for PCIe working with VFIO (user space driver) is investigated in this repository.

* VFIO: Virtual Function I/O (https://www.kernel.org/doc/html/latest/driver-api/vfio.html)
* Xilinx DMA: Xilinx DMA/bridge subsystem for PCI Express (PCIe) implements a high performance, configurable scatter gather DMA for use with the PCI Express 2.1 and 3.x integrated block (https://www.xilinx.com/support/documentation/ip_documentation/xdma/v4_1/pg195-pcie-dma.pdf)

The idea is inspired by the repository NVIDIA jetson-rdma-picoevb 
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

// XLNX_REG(C2H, 0, H2C_CTRL) =
// (( XLNX_DMA_TARGET_C2H << 12 ) | \
// (( 0 ) << 8) | \
// XLNX_DMA_H2C_CTRL )
```

# RDMA control (Xilinx PG195)

## Descriptors (or structure fields)

## SG-DMA initialization