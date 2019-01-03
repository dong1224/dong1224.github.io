# qemu内存结构MemoryRegion和AddressSpace

## RAMBLOCK

RAMBLOCK才是真正分配了host内存的地方，但实际它上不仅仅代表内存，还有设备自有内存，显存。
它的主要元素就是mr，host, offset和used_length。

	struct RAMBlock {
	 struct rcu_head rcu;
	 struct MemoryRegion *mr;
	 uint8_t *host;
	 ram_addr_t offset;
	 ram_addr_t used_length;
	 ram_addr_t max_length;
	 void (*resized)(const char*, uint64_t length, void *host);
	 uint32_t flags;
	 /* Protected by iothread lock. */
	 char idstr[256];
	 /* RCU-enabled, writes protected by the ramlist lock */
	 QLIST_ENTRY(RAMBlock) next;
	 int fd;
	 size_t page_size;
	};
	
每个RAMBLOCK都有一个唯一的MemoryRegion对应，另外需要注意的是不是每个MemoryRegion都有RAMBLOCK对应的。
host则是host的线性地址。

	new_block->host = phys_mem_alloc(new_block->max_length, &new_block->mr->align);

offset则是这个block在ram中的base地址，如qemu_get_ram_block下的代码

	block = atomic_rcu_read(&ram_list.mru_block);
	 if (block && addr - block->offset < block->max_length) {
	 return block;
	}
	
## MemoryRegion

	struct MemoryRegion {
	 Object parent_obj;

	/* All fields are private - violators will be prosecuted */

	/* The following fields should fit in a cache line */
	 bool romd_mode;
	 bool ram;
	 bool subpage;
	 bool readonly; /* For RAM regions */
	 bool rom_device;
	 bool flush_coalesced_mmio;
	 bool global_locking;
	 uint8_t dirty_log_mask;
	 RAMBlock *ram_block;
	 Object *owner;
	 const MemoryRegionIOMMUOps *iommu_ops;

	const MemoryRegionOps *ops;
	 void *opaque;
	 MemoryRegion *container;
	 Int128 size;
	 hwaddr addr;
	 void (*destructor)(MemoryRegion *mr);
	 uint64_t align;
	 bool terminates;
	 bool ram_device;
	 bool enabled;
	 bool warning_printed; /* For reservations */
	 uint8_t vga_logging_count;
	 MemoryRegion *alias;
	 hwaddr alias_offset;
	 int32_t priority;
	 QTAILQ_HEAD(subregions, MemoryRegion) subregions;
	 QTAILQ_ENTRY(MemoryRegion) subregions_link;
	 QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) coalesced;
	 const char *name;
	 unsigned ioeventfd_nb;
	 MemoryRegionIoeventfd *ioeventfds;
	 QLIST_HEAD(, IOMMUNotifier) iommu_notify;
	 IOMMUNotifierFlag iommu_notify_flags;
	};

MemoryRegion是树状父子结构的，每一个ramblock都有一个对应的MemoryRegion，
一般而言，这个MemoryRegion是最顶级的MemoryRegion，它还有很多子MemoryRegion，
比如在这个ramblock地址范围内的MMIO等，这里面的重点元素是size，addr和alias_offset，

	memory_region_init_alias(ram_below_4g, NULL, "ram-below-4g", ram, 0, pcms->below_4g_mem_size);

	memory_region_init(mr, owner, name, size);

	mr->size = int128_make64(size);

	mr->alias_offset = offset;
	
addr的由来则是

	memory_region_add_subregion(system_memory, 0x100000000ULL,ram_above_4g);

	memory_region_add_subregion_common(mr, offset, subregion);

	subregion->addr = offset;
	
此处的addr实际也是mr的起始地址，和offset有点类似

	struct AddressSpace {
	 /* All fields are private. */
	 struct rcu_head rcu;
	 char *name;
	 MemoryRegion *root;
	 int ref_count;
	 bool malloced;

	/* Accessed via RCU. */
	 struct FlatView *current_map;

	int ioeventfd_nb;
	 struct MemoryRegionIoeventfd *ioeventfds;
	 struct AddressSpaceDispatch *dispatch;
	 struct AddressSpaceDispatch *next_dispatch;
	 MemoryListener dispatch_listener;
	 QTAILQ_HEAD(memory_listeners_as, MemoryListener) listeners;
	 QTAILQ_ENTRY(AddressSpace) address_spaces_link;
	};
	
## AddressSpace

struct AddressSpace {
 /* All fields are private. */
 struct rcu_head rcu;
 char *name;
 MemoryRegion *root;
 int ref_count;
 bool malloced;

/* Accessed via RCU. */
 struct FlatView *current_map;

int ioeventfd_nb;
 struct MemoryRegionIoeventfd *ioeventfds;
 struct AddressSpaceDispatch *dispatch;
 struct AddressSpaceDispatch *next_dispatch;
 MemoryListener dispatch_listener;
 QTAILQ_HEAD(memory_listeners_as, MemoryListener) listeners;
 QTAILQ_ENTRY(AddressSpace) address_spaces_link;
};

地址空间，本来不同的设备使用的地址空间不同，但是QEMU X86里面只有两种，
address_space_memory和address_space_io，所有设备的地址空间都被映射到了这两个上面，

	struct FlatRange {
	 MemoryRegion *mr;
	 hwaddr offset_in_region;
	 AddrRange addr;
	 uint8_t dirty_log_mask;
	 bool romd_mode;
	 bool readonly;
	};

	struct FlatView {
	 struct rcu_head rcu;
	 unsigned ref;
	 FlatRange *ranges;
	 unsigned nr;
	 unsigned nr_allocated;
	};
	
是通过FlatView体现的，当memory region发生变化的时候，
执行memory_region_transaction_commit，address_space_update_topology，
address_space_update_topology_pass最终完成更新FlatView的目标。

FlatView结构如下

![2.1](https://github.com/dong1224/dong1224.github.io/blob/master/_posts/201901/2.1.png?raw=true)

每一个flat range都投射到同一个地址空间的平面上，而上图中的R1，R2等对应的则是struct MemoryRegionSection。

	struct MemoryRegionSection {
	 MemoryRegion *mr;
	 AddressSpace *address_space;
	 hwaddr offset_within_region;
	 Int128 size;
	 hwaddr offset_within_address_space;
	 bool readonly;
	};
	
在下面被调用

	#define MEMORY_LISTENER_UPDATE_REGION(fr, as, dir, callback, _args...) \
	 do { \
	 MemoryRegionSection mrs = section_from_flat_range(fr, as); \
	 MEMORY_LISTENER_CALL(as, callback, dir, &mrs, ##_args); \
	 } while(0)
	 
在MEMORY_LISTENER_CALL中调用kvm_region_add和kvm_region_del，执行kvm_set_phys_mem，组装KVMSlot

	typedef struct KVMSlot
	{
	 hwaddr start_addr;
	 ram_addr_t memory_size;
	 void *ram;
	 int slot;
	 int flags;
	} KVMSlot;
	
最终通过kvm_userspace_memory_region将QEMU的内存分布信息传递给KVM。

具体元素的关系，见下图

![2.2](https://github.com/dong1224/dong1224.github.io/blob/master/_posts/201901/2.2.jpg?raw=true)