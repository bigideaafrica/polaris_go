# VM Setup Guide for Polaris Subnet

## Introduction

This document outlines the process, requirements, and costs associated with migrating the Polaris subnet from a container-based architecture to a virtual machine (VM) based solution. It includes technical specifications, implementation steps, cost analysis, and recommendations.

## Container vs. VM Comparison

| Aspect | Containers (Current) | Virtual Machines (Proposed) |
|--------|---------------------|----------------------------|
| Resource Usage | Lightweight, shared kernel | Heavyweight, full OS per instance |
| Isolation | Process-level isolation | Hardware-level isolation |
| Startup Time | Seconds | Minutes |
| Density | High (dozens per host) | Low (5-10 per host) |
| Flexibility | Limited to host OS family | Any OS supported by hypervisor |
| Security | Good, but shared kernel vulnerabilities | Strong isolation between VMs |
| Cost | Low resource overhead | High resource requirement |

## Hardware Requirements

### Minimum Server Specifications (Per Host)

For a production environment hosting 10-20 simultaneous development VMs:

* **CPU**: 16+ cores (32+ threads recommended)
* **RAM**: 64GB+ (128GB recommended)
* **Storage**: 1TB+ SSD (NVMe preferred)
* **Network**: 1Gbps minimum (10Gbps recommended)

### Client Requirements

* **CPU**: Support for hardware virtualization (Intel VT-x/AMD-V)
* **OS**: Linux with kernel 5.0+ (for optimal KVM support)

## Software Requirements

### Core Components

1. **Hypervisor**:
   * KVM/QEMU (recommended for Linux)
   * VMware ESXi (commercial alternative)
   * Hyper-V (for Windows Server environments)

2. **Management Layer**:
   * libvirt + libvirt-python
   * OpenStack (for larger deployments)
   * VMware vCenter (for ESXi)

3. **Programming Dependencies**:
   * Python 3.8+
   * libvirt-python
   * paramiko (SSH library)
   * pyvmomi (for VMware)

4. **VM Image Management**:
   * qemu-img
   * cloud-init for VM customization

## Implementation Steps

### 1. Hypervisor Setup

```bash
# Install KVM and libvirt on Ubuntu/Debian
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst

# Enable and start libvirt service
sudo systemctl enable libvirtd
sudo systemctl start libvirtd

# Add current user to libvirt group
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
```

### 2. Create Base VM Template

```bash
# Download Ubuntu Server image
wget https://releases.ubuntu.com/22.04/ubuntu-22.04-live-server-amd64.iso

# Create base VM image
sudo virt-install --name polaris-base-template \
  --memory 2048 \
  --vcpus 2 \
  --disk size=20 \
  --network bridge=virbr0 \
  --os-variant ubuntu22.04 \
  --cdrom ubuntu-22.04-live-server-amd64.iso

# After installation, configure the VM with necessary dev tools
# Then shut down the VM and create a template
sudo virt-sysprep -d polaris-base-template --enable customize --firstboot-command \
  'apt-get update && apt-get install -y python3-pip git openssh-server'
```

### 3. Python Code Implementation

Create `vm_manager.py` in the `/home/tang/polaris_go/polaris-subnet/src/` directory:

```python
import libvirt
import uuid
import logging
import time
import xml.etree.ElementTree as ET
from pathlib import Path

class VMManager:
    def __init__(self):
        """Initialize connection to hypervisor."""
        self.logger = logging.getLogger(__name__)
        self.conn = libvirt.open('qemu:///system')
        if not self.conn:
            raise Exception("Failed to connect to hypervisor")
        
        self.base_template = "polaris-base-template"
        self.vm_storage_path = "/var/lib/libvirt/images/"
        
    def create_vm(self, name=None):
        """Create a new VM from the base template."""
        if not name:
            name = f"polaris-dev-{str(uuid.uuid4())[:8]}"
            
        # Clone from base template
        source_dom = self.conn.lookupByName(self.base_template)
        source_xml = source_dom.XMLDesc(0)
        
        # Modify XML to create new VM
        tree = ET.fromstring(source_xml)
        tree.find('.//name').text = name
        
        # Update UUID
        new_uuid_elem = tree.find('.//uuid')
        new_uuid_elem.text = str(uuid.uuid4())
        
        # Update disk paths
        disk_elem = tree.find('.//disk[@device="disk"]')
        source_elem = disk_elem.find('.//source')
        old_path = source_elem.get('file')
        new_path = f"{self.vm_storage_path}{name}.qcow2"
        source_elem.set('file', new_path)
        
        # Create disk clone
        import subprocess
        subprocess.run([
            'qemu-img', 'create', '-f', 'qcow2', 
            '-b', old_path, new_path
        ])
        
        # Define and start new VM
        new_xml = ET.tostring(tree).decode()
        new_dom = self.conn.defineXML(new_xml)
        new_dom.create()
        
        # Wait for VM to get IP address
        max_wait = 120  # seconds
        start_time = time.time()
        ip_address = None
        
        while time.time() - start_time < max_wait:
            if new_dom.isActive():
                try:
                    ifaces = new_dom.interfaceAddresses(libvirt.VIR_DOMAIN_INTERFACE_ADDRESSES_SRC_LEASE)
                    for _, iface_data in ifaces.items():
                        for addr in iface_data['addrs']:
                            if addr['type'] == libvirt.VIR_IP_ADDR_TYPE_IPV4:
                                ip_address = addr['addr']
                                break
                except libvirt.libvirtError:
                    pass
                    
            if ip_address:
                break
                
            time.sleep(5)
            
        if not ip_address:
            self.logger.error(f"Could not determine IP address for VM {name}")
            return None
            
        return {
            "vm_id": name,
            "ip_address": ip_address,
            "ssh_port": 22
        }
        
    def delete_vm(self, vm_id):
        """Delete a VM."""
        try:
            dom = self.conn.lookupByName(vm_id)
            if dom.isActive():
                dom.destroy()  # Force shutdown
            dom.undefine()
            
            # Remove disk
            disk_path = f"{self.vm_storage_path}{vm_id}.qcow2"
            import os
            if os.path.exists(disk_path):
                os.remove(disk_path)
                
            return True
        except libvirt.libvirtError as e:
            self.logger.error(f"Failed to delete VM {vm_id}: {str(e)}")
            return False
```

### 4. Update Compute Subnet API

Modify `/home/tang/polaris_go/polaris-subnet/compute_subnet/src/services/container.py` to use VMs instead of containers.

### 5. Update Installation Scripts

Modify installation scripts to include hypervisor setup and VM template creation.

## Cost Analysis

### Hardware Costs

| Component | Containers (Current) | VMs (Proposed) | Difference |
|-----------|----------------------|----------------|------------|
| CPU cores | 4-8 cores | 16-32 cores | 4x increase |
| RAM | 16-32GB | 64-128GB | 4x increase |
| Storage | 250GB-500GB | 1TB-2TB | 4x increase |
| Server Hardware | $1,000-$2,000 | $4,000-$8,000 | 4x increase |

### Software Costs

| Component | Containers (Current) | VMs (Open Source) | VMs (Commercial) |
|-----------|----------------------|-------------------|------------------|
| Core Platform | Docker (Free) | KVM/QEMU (Free) | VMware ESXi ($$$) |
| Management | Docker Compose (Free) | libvirt (Free) | vCenter ($$$) |
| Orchestration | Kubernetes (Free) | OpenStack (Free) | vRealize ($$$) |
| Total Software | $0 | $0 | $1,000-$5,000+ |

### Cloud Costs (Monthly, if hosted in cloud)

| Resource | Containers | VMs | Difference |
|----------|------------|-----|------------|
| 10 instances | $100-$200 | $500-$1,000 | 5x increase |
| 50 instances | $500-$1,000 | $2,500-$5,000 | 5x increase |
| 100 instances | $1,000-$2,000 | $5,000-$10,000 | 5x increase |

### Development Costs

* **Engineering Time**: 80-160 hours to refactor codebase
* **Testing**: 40-80 hours for testing and validation
* **Documentation**: 20-40 hours for documentation updates
* **Total Dev Cost**: Approximately $14,000-$28,000 (at $100/hr)

### Performance Costs

* **Provisioning Time**: Containers (seconds) vs VMs (minutes)
* **Resource Utilization**: 3-5x more hardware resources needed for VMs
* **User Capacity**: 75-80% reduction in total users per host

## Recommendations

### Recommendation: Continue Using Containers

**I recommend against migrating to VMs for the following reasons:**

1. **Cost Efficiency**: Containers provide a 3-5x cost advantage over VMs in both hardware and operational expenses.

2. **Performance**: Containers offer faster startup times and lower latency, which is crucial for development environments.

3. **Scalability**: The current container architecture can scale to support more users without significant hardware investments.

4. **Development Effort**: The engineering effort to refactor the system would be substantial and likely not deliver enough benefits to justify the cost.

### Alternative Improvements to Consider

Instead of a full VM migration, consider these container improvements:

1. **Enhanced Isolation**: Implement stronger container isolation using features like:
   * Seccomp profiles
   * AppArmor/SELinux policies
   * User namespace isolation
   * Resource limits (cgroups)

2. **Hybrid Approaches**: For specific use cases requiring stronger isolation:
   * **Kata Containers**: VM-like security with container-like management
   * **gVisor**: Container runtime with an additional security layer
   * **Firecracker MicroVMs**: Lightweight VMs with fast startup times

3. **Storage Optimization**: Improve data persistence with named volumes, bind mounts, and backup strategies.

4. **Networking Enhancements**: Implement network policies for better security isolation between containers.

### When VMs Would Make Sense

VMs would only be justified if:

1. **Regulatory Requirements**: Your compliance framework specifically requires hardware-level isolation
2. **Operating System Diversity**: Users need to run non-Linux operating systems
3. **Kernel Modifications**: Users need to modify kernel parameters or load custom kernel modules
4. **Legacy Applications**: Supporting applications that cannot be containerized

## Conclusion

While migrating to VMs is technically feasible, the significant cost increases (3-5x) and performance penalties make it difficult to recommend for the Polaris subnet. The current container-based architecture provides a more cost-effective and efficient solution for development environments.

If specific security or isolation concerns exist, consider the hybrid approaches mentioned above before a full VM migration. 