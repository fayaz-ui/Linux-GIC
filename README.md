# Linux Generic Interrupt Controller (GIC) Guide

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Linux Kernel](https://img.shields.io/badge/Linux-Kernel-blue.svg)](https://www.kernel.org/)
[![ARM](https://img.shields.io/badge/Architecture-ARM-green.svg)](https://www.arm.com/)

> A comprehensive guide to understanding Linux Generic Interrupt Controller (GIC) architecture, APIs, and device driver integration.

## 📖 Overview

The Generic Interrupt Controller (GIC) is ARM's standard interrupt controller used in modern ARM-based systems. This repository provides detailed documentation, examples, and flow diagrams explaining how the GIC integrates with the Linux kernel and device drivers.

## 🏗️ Architecture Overview

### 5 Main Layers

```
┌─────────────────────────────────────────────────────────┐
│                    5. User Space                        │
│           Final notification/completion                 │
└─────────────────────────────────────────────────────────┘
                              ⬆️
┌─────────────────────────────────────────────────────────┐
│                4. Device Driver Layer                   │
│           Application-specific handlers                 │
└─────────────────────────────────────────────────────────┘
                              ⬆️
┌─────────────────────────────────────────────────────────┐
│               3. Generic IRQ Subsystem                  │
│              Linux abstraction layer                   │
└─────────────────────────────────────────────────────────┘
                              ⬆️
┌─────────────────────────────────────────────────────────┐
│                2. GIC Driver Layer                      │
│          Hardware-specific interrupt handling          │
└─────────────────────────────────────────────────────────┘
                              ⬆️
┌─────────────────────────────────────────────────────────┐
│                 1. Hardware Layer                       │
│              Physical GIC and devices                  │
└─────────────────────────────────────────────────────────┘
```

## 🚀 Key Features

- **Hardware Abstraction**: GIC provides unified interrupt handling across ARM platforms
- **Multi-core Support**: Distributes interrupts across CPU cores efficiently  
- **Priority Management**: Hardware-based interrupt prioritization
- **Power Management**: Supports CPU idle states and power gating
- **Virtualization**: Hardware support for virtualized interrupt handling

## 🔧 GIC Versions

| Version | Description | Key Features |
|---------|-------------|--------------|
| **GIC-400** | GICv2 implementation | Up to 8 cores, 1020 interrupts |
| **GIC-500** | GICv3 implementation | Scalable, affinity routing |
| **GIC-600** | Latest GICv3 | Enhanced performance, security |

## 📋 Interrupt Types

```c
// Interrupt Types in GIC
#define SGI_INTERRUPT    0-15     // Software Generated Interrupts
#define PPI_INTERRUPT    16-31    // Private Peripheral Interrupts  
#define SPI_INTERRUPT    32-1019  // Shared Peripheral Interrupts
```

### Interrupt Categories

- **SGI (0-15)**: Inter-processor communication
- **PPI (16-31)**: Per-CPU private interrupts (timers, etc.)
- **SPI (32-1019)**: Shared device interrupts

## 🔄 Interrupt Flow

### Complete Processing Flow

1. **Hardware Interrupt** → Device asserts signal to GIC
2. **GIC Processing** → Distributor routes to CPU interface
3. **CPU Exception** → CPU receives IRQ/FIQ exception
4. **Low-level Handler** → Assembly saves context, calls `gic_handle_irq()`
5. **IRQ Acknowledgment** → Read GIC IAR register
6. **Generic IRQ Layer** → Convert HW IRQ to Linux IRQ
7. **Flow Handler** → Call appropriate handler type
8. **Device Handler** → Execute driver's interrupt handler
9. **EOI (End of Interrupt)** → Signal completion to GIC
10. **Context Restoration** → Return from interrupt

## 🛠️ Core APIs

### Driver Registration
```c
// Register interrupt handler
int request_irq(unsigned int irq, irq_handler_t handler, 
                unsigned long flags, const char *name, void *dev);

// Device-managed registration (recommended)
int devm_request_irq(struct device *dev, unsigned int irq,
                     irq_handler_t handler, unsigned long irqflags,
                     const char *devname, void *dev_id);

// Free interrupt handler
void free_irq(unsigned int irq, void *dev_id);
```

### IRQ Control
```c
// Disable/Enable interrupts
void disable_irq(unsigned int irq);
void enable_irq(unsigned int irq);

// Set CPU affinity
int irq_set_affinity(unsigned int irq, const struct cpumask *cpumask);
```

### Platform Device Integration
```c
// Get IRQ from platform device
int platform_get_irq(struct platform_device *dev, unsigned int num);

// Get IRQ from device tree
int of_irq_get(struct device_node *dev, int index);
```

## 📝 Example Device Driver

```c
#include <linux/interrupt.h>
#include <linux/platform_device.h>

struct my_device {
    void __iomem *base;
    int irq;
    struct device *dev;
};

static irqreturn_t my_device_irq_handler(int irq, void *data)
{
    struct my_device *mydev = (struct my_device *)data;
    u32 status;
    
    // Read interrupt status
    status = readl(mydev->base + STATUS_REG);
    
    if (!(status & IRQ_PENDING))
        return IRQ_NONE;
    
    // Handle interrupt
    dev_info(mydev->dev, "Interrupt handled: 0x%x\n", status);
    
    // Clear interrupt
    writel(status, mydev->base + STATUS_REG);
    
    return IRQ_HANDLED;
}

static int my_device_probe(struct platform_device *pdev)
{
    struct my_device *mydev;
    int ret;
    
    mydev = devm_kzalloc(&pdev->dev, sizeof(*mydev), GFP_KERNEL);
    if (!mydev)
        return -ENOMEM;
    
    // Get IRQ number from device tree
    mydev->irq = platform_get_irq(pdev, 0);
    if (mydev->irq < 0)
        return mydev->irq;
    
    // Request interrupt
    ret = devm_request_irq(&pdev->dev, mydev->irq,
                          my_device_irq_handler,
                          IRQF_TRIGGER_RISING,
                          "my-device", mydev);
    if (ret)
        return ret;
    
    platform_set_drvdata(pdev, mydev);
    return 0;
}

static const struct of_device_id my_device_of_match[] = {
    { .compatible = "vendor,my-device" },
    {},
};

static struct platform_driver my_device_driver = {
    .probe = my_device_probe,
    .driver = {
        .name = "my-device",
        .of_match_table = my_device_of_match,
    },
};

module_platform_driver(my_device_driver);
```

## 🌳 Device Tree Configuration

```dts
/ {
    interrupt-controller@8000000 {
        compatible = "arm,gic-400";
        #interrupt-cells = <3>;
        interrupt-controller;
        reg = <0x8001000 0x1000>,    /* Distributor */
              <0x8002000 0x2000>;    /* CPU Interface */
    };
    
    my-device@12345000 {
        compatible = "vendor,my-device";
        reg = <0x12345000 0x1000>;
        interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-parent = <&gic>;
    };
};
```

## 🔍 Debugging Tools

### Kernel Debug Information
```bash
# View interrupt statistics
cat /proc/interrupts

# View IRQ affinity
cat /proc/irq/*/smp_affinity

# Enable IRQ debugging
echo 1 > /sys/kernel/debug/tracing/events/irq/enable
```

### GIC-specific Debug
```bash
# GIC distributor registers (if debugfs mounted)
cat /sys/kernel/debug/irq/gics/gic*/distributor

# Check IRQ domain mappings
cat /sys/kernel/debug/irq/domains/*
```

## 🚨 Common Issues and Solutions

### Issue: Interrupt Not Triggering
```c
// Check trigger type in device tree
interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;

// Verify in driver
ret = devm_request_irq(&pdev->dev, mydev->irq,
                      my_device_irq_handler,
                      IRQF_TRIGGER_RISING,  // Must match DT
                      "my-device", mydev);
```

### Issue: Shared Interrupt Conflicts
```c
// Use IRQF_SHARED for shared interrupts
ret = request_irq(irq, handler, IRQF_SHARED, "device", dev_id);

// Handler must check if interrupt is from this device
static irqreturn_t handler(int irq, void *data) {
    if (!is_my_interrupt())
        return IRQ_NONE;  // Not our interrupt
    
    // Handle interrupt
    return IRQ_HANDLED;
}
```

## 📊 Performance Considerations

### IRQ Affinity Optimization
```c
// Set interrupt to specific CPU
cpumask_t mask;
cpumask_clear(&mask);
cpumask_set_cpu(2, &mask);  // CPU 2
irq_set_affinity(irq, &mask);
```

### Threaded Interrupts
```c
// Use threaded IRQ for heavy processing
ret = request_threaded_irq(irq, 
                          quick_handler,      // Quick handler (hard IRQ)
                          threaded_handler,   // Threaded handler
                          IRQF_ONESHOT,
                          "device", dev);
```

## 📚 References

- [ARM Generic Interrupt Controller Architecture Specification](https://developer.arm.com/documentation/ihi0069/latest/)
- [Linux Kernel IRQ Subsystem Documentation](https://www.kernel.org/doc/html/latest/core-api/genericirq.html)
- [Device Tree Interrupt Mapping](https://www.kernel.org/doc/Documentation/devicetree/bindings/interrupt-controller/interrupts.txt)

---

⭐ **Star this repository** if you found it helpful!

**Made with ❤️ for the Linux kernel community**
