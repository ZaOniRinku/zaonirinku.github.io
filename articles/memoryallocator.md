# A simple Vulkan Memory Allocator
Allocating memory on the device (GPU) is a mandatory part for a Vulkan application, either to allocate images that your application is going to use as textures or render targets or buffers to pass information to your shaders or write on it.

This article is aimed to explain how to write a simple Memory Allocator for a Vulkan application.

## The importance of a Memory Allocator
With Vulkan, you can allocate memory on the GPU using the **[vkAllocateMemory](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkAllocateMemory.html)** function. This function will allocate a certain amount of **[a certain type of memory](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkMemoryType.html)** and will **decrease the number of allocations your GPU driver allows you to do**.

![Maximum number of memory allocations for a Nvidia GeForce GTX 1060 6GB (driver version 471.22.0.0)](https://i.imgur.com/vdR2yIV.png)
<sub>Maximum number of memory allocations for a Nvidia GeForce GTX 1060 6GB (driver version 471.22.0.0) - [vulkan.gpuinfo.org](https://vulkan.gpuinfo.org)</sub>

If you reach the maximum number of allocations by calling vkAllocateMemory too many times, your application won't be able to allocate any more device memory. That can happen if you decide to allocate device memory each time you want to load an image or a buffer. For example, if we want to load an object with a mesh and metallic-roughness PBR workflow with a diffuse map, a normal map, a metallic map, a roughness map, an occlusion map and an emissive map, we would have 1 buffer and 6 images. If we allocate memory once per resource, **that would make us allocate 7 times to load a single object**. If we have 4096 possible maximum allocations, that would mean that **we would only be able to load 585 objects, not counting uniform buffer objects and render targets**.
The goal here will be **to allocate a big chunk of memory** and bind multiple **[images](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkBindImageMemory.html)** and **[buffers](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkBindBufferMemory.html)** on a single chunk of memory. From now on, **"allocating" means that we bind objects, not allocate a new device memory**.

## Writing your own Memory Allocator
In a real application, it is recommended to use the industry-standard **[AMD's Vulkan Memory Allocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)** (note: even if it's made by AMD, it works with any constructor's devices). The allocator described in this article **will be really simple** and **will not cover every edge cases that VMA (Vulkan Memory Allocator) probably handles very well**. Making a memory allocator is still a pretty good exercise and can allow you to understand GPU memory management better.

## Memory Allocator
This allocator is the one used in **[NeigeEngine](https://github.com/ZaOniRinku/NeigeEngine/blob/main/src/utils/memoryallocator/MemoryAllocator.h)** and may change in the future. 

### Global principles
 There are 3 levels : the **Memory Allocator** itself, which contains a vector of chunks. A **Chunk** is an allocation with **vkAllocateMemory**, by default, its size will be 256MB and contains a doubly linked list of blocks. A **Block** is an image or buffer memory inside a chunk.

![The structure of this memory allocator](https://i.imgur.com/oWwb7kb.png)<sub>The structure of this memory allocator</sub>

The doubly linked list allows for **an easy way to add and remove elements in the list**. When we allocate a new image or buffer, we want **to subdivide a block into two blocks**: one that is going to be used to represent the allocated resource, and a new block that represents the remaining memory of the previous block.

![Block subvision](https://i.imgur.com/5aMbAW9.png)
<sub>Block subvision</sub>

When we want to deallocate a resource, we just change the flag **inUse** from *true* to *false*, then we check if the previous block is in use, if it's not, we fuse them together, taking the minimum offset and adding the sizes. We also do the same thing for the next block. It's also important to take care on how we re-link the blocks.
### Data structures
We need 3 data structures : one for the Memory Allocator itself, one for the chunks and one for the blocks.
Let's start with the Block structure. It needs **a pointer to a previous and a pointer to a next block** as it's going to be a doubly linked list. The advantage to use a doubly linked list is that it's way easier to insert and delete random elements in the list than when using a vector or an array.
It then needs to keep track of its **offset inside a chunk and its size**.
There is also an **allocation index** that is going to be used when we need to destroy an image or a buffer.
Finally, there is a boolean telling us if this Block is **in use** at the moment or not.
```CPP
struct Block {
	Block* prev;
	Block* next;

	VkDeviceSize offset;
	VkDeviceSize size;
	VkDeviceSize allocationId;
	bool inUse;
}
```

The Chunk structure has an **index**, also for deallocation.
It has a VkDeviceMemory, which is **the actual memory object**, a **memory type**, because **we cannot put resources with different memory types in the same chunk of memory**, and finally **a pointer to a block, that is the head of the doubly linked list**.
This structure also has functions: a constructor, a function to "allocate" (which actually just binds a resource to the chunk) and a resource to free all of this chunk's blocks.
```CPP
struct Chunk {
	VkDeviceSize id;
	VkDeviceMemory memory;
	int32_t type;
	Block* head;

	Chunk(VkDeviceSize chunkId, int32_t memoryType, VkDeviceSize size);
	VkDeviceSize allocate(VkMemoryRequirements memRequirements, VkDeviceSize* allocationNumber, MemoryInfo* memoryInfo);
	void freeBlocks();
}
```

The **MemoryInfo** structure is another structure, external to the Memory Allocator, which allows to keep some information about a resource and its memory.
The goal of this structure is to have a fast way to find the right block for an image or a buffer when it's time to deallocate it. **It is not mandatory** as there are other ways to find on which chunks a resource is allocated.
```CPP
struct MemoryInfo {
	VkDeviceSize chunkId;
	VkDeviceSize offset;
	VkDeviceSize allocationId;
}
```

Finally, the Memory Allocator structure is the highest-level one, it's the one we need in other parts of the Vulkan renderer, to allocate memory.
It has a list of chunks, here, a vector from C++'s standard library, and a way to keep track of the number of resources that have been bound.

Some of these functions are the ones that are going to be called in the renderer. **destroy** is used to completely wipe the device's memory when the application closes.
**allocate** is an overloaded function, one is used with images and the other for buffers.
**deallocate** allows to deallocate a block.
**findProperties** is a function to find an optimal memory type with all the required properties. This one is given in **[Vulkan's specification](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html)** (search for *findProperties*).
```CPP
struct MemoryAllocator {
	std::vector<Chunk> chunks;
	VkDeviceSize allocationNumber = 1;

	void destroy();
	VkDeviceSize allocate(VkBuffer* bufferToAllocate, VkMemoryPropertyFlags flags, MemoryInfo* memoryInfo);
	VkDeviceSize allocate(VkImage* imageToAllocate, VkMemoryPropertyFlags flags, MemoryInfo* memoryInfo);
	void deallocate(VkDeviceSize chunkId, VkDeviceSize allocationId);
	int32_t findProperties(uint32_t memoryTypeBitsRequirement, VkMemoryPropertyFlags requiredProperties);
}
```

It is now time to write the functions.

### Memory Allocator functions
The **findProperties** function is given in Vulkan's specification, it consists of a for loop in the different types of memories to find a suitable one, given the properties the allocation requires. 
```CPP
// Find a memory in `memoryTypeBitsRequirement` that includes all of `requiredProperties`
int32_t findProperties(const VkPhysicalDeviceMemoryProperties* pMemoryProperties,
                       uint32_t memoryTypeBitsRequirement,
                       VkMemoryPropertyFlags requiredProperties) {
    const uint32_t memoryCount = pMemoryProperties->memoryTypeCount;
    for (uint32_t memoryIndex = 0; memoryIndex < memoryCount; ++memoryIndex) {
        const uint32_t memoryTypeBits = (1 << memoryIndex);
        const bool isRequiredMemoryType = memoryTypeBitsRequirement & memoryTypeBits;

        const VkMemoryPropertyFlags properties =
            pMemoryProperties->memoryTypes[memoryIndex].propertyFlags;
        const bool hasRequiredProperties =
            (properties & requiredProperties) == requiredProperties;

        if (isRequiredMemoryType && hasRequiredProperties)
            return static_cast<int32_t>(memoryIndex);
    }

    // failed to find memory type
    return -1;
}
```
<sub>Function written by Kronos Group - https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html</sub>

**The allocate functions are the ones the rendered needs to call when it needs to allocate memory for an image or a buffer**. It is iterating through the chunk's list to find a chunk with the right memory type and a block with enough memory to put the resource we want to allocate. If it does not find a single chunk that fills these requirements, it creates a new one. It then binds  the resource to the chunk.
```CPP
VkDeviceSize MemoryAllocator::allocate(VkImage* imageToAllocate, VkMemoryPropertyFlags flags, MemoryInfo* memoryInfo) {
	VkMemoryRequirements memRequirements;
	vkGetImageMemoryRequirements(logicalDevice, *imageToAllocate, &memRequirements);
	int32_t properties = findProperties(memRequirements.memoryTypeBits, flags);
	if (properties == -1) { // Unable to find suitable memory type.
		exit(1);
	}

	// Look for the first block with enough space
	for (Chunk& chunk : chunks) {
		if (chunk.type == properties) { // Same properties
			VkDeviceSize offset;
			offset = chunk.allocate(memRequirements, &allocationNumber, memoryInfo);

			if (offset != -1) {
				vkBindImageMemory(logicalDevice, *imageToAllocate, chunk.memory, offset); // Bind the resource
				return allocationNumber - 1;
			}
		}
	}
	
	// No block has been found, create a new chunk
	Chunk newChunk = Chunk(static_cast<VkDeviceSize>(chunks.size()), properties, std::max((VkDeviceSize)CHUNK_SIZE, memRequirements.size)); // CHUNK_SIZE is 268435456 (256MB) by default, but we want to be able to allocate resources even bigger than that

	// Bind the resource to this chunk
	VkDeviceSize offset;
	offset = newChunk.allocate(memRequirements, &allocationNumber, memoryInfo);

	if (offset == -1) { // Unable to allocate memory for whatever reason
		exit(1);
	}

	vkBindImageMemory(logicalDevice, *imageToAllocate, newChunk.memory, offset);

	chunks.push_back(newChunk);

	return allocationNumber - 1;
}
```

This function allocates images but allocating buffers is the exact same thing, you just need to replace Image by Buffer.

The deallocate function uses some information contain in the MemoryInfo structure. The goal is to access the right block in the right chunk as quick as possible. It is indeed possible to use less information and still be able to find the right block, but that would mean to look at every chunk with the right type and iterate through the entire block doubly linked list to find the right block.
When the right block has been found, we need to put the **inUse boolean to false**, check if the previous block is in use and if it's not, fuse them. Then do the same with the next block.

![A deallocation](https://i.imgur.com/R3bKViT.png)<sub>A deallocation</sub>

```CPP
void MemoryAllocator::deallocate(VkDeviceSize chunkId, VkDeviceSize allocationId) {
	Chunk* chunk = &chunks[chunkId]; // We directly go to the right chunk
	Block* curr = chunk->head; // Get the head of the doubly linked list
	while (curr) {
		if (curr->inUse && curr->allocationId = allocationId) {
			curr->inUse = false; // The block is not in use anymore
			curr->allocationId = 0; // Reset the allocation id

			// Block fusion
			// Fusion with previous block
			if (curr->prev && !curr->prev->inUse) { // If the current block is not the head and the previous block is not in use
				Block* prev = curr->prev;
				if (prev->prev) { // If the previous block is not the head
					prev->prev->next = curr; // The next block of the previous block of the previous block is the current block
					curr->prev = prev->prev; // The previous block of the current block is the previous block of the previous block
				}
				else { // Else, the previous block is the head
					curr->prev = nullptr;
					chunk->head = curr; // The current block becomes the head
				}
				curr->offset = prev->offset; // The offset of the current block is the offset of the previous block
				curr->size += prev->size; // We add the previous block's size to the current block's size
				delete prev; // Destroy the previous block;
			}
			// Fusion with next block
			if (curr->next && !curr->next->inUse) { // If the current block is not the tail and the next block is not in use
				Block* next = curr->next;
				if (next->next) { // If the next block is not the tail
					next->next->prev = curr; // The previous block of the next block of the next bloock is the current block
					curr->next = next->next; // The next block of the current block is the next block of the next block
				}
				else { // Else, the next block is the tail
					curr->next = nullptr;
				}
				curr->size += next->size; // We add the next block's size to the current block's size
				delete next; // Destroy the next block;
			}
			return;
		}
		curr = curr->next; // Go to the next block
	}
	// If the program gets to this point, then the deallocation failed
}
```

The destroy functions frees all blocks in all chunks and frees the memory of all chunks.
```CPP
void MemoryAllocator::destroy() {
	chunk.freeBlocks();
	vkFreeMemory(logicalDevice, chunk.memory, nullptr);
}
```

### Chunk functions
The constructor allows us to create a new chunk with the right properties. It is also here that we call **vkAllocateMemory**.
```CPP
Chunk::Chunk(VkDeviceSize chunkId, int32_t memoryType, VkDeviceSize size) {
	id = chunkId;
	type = memoryType;
	
	VkMemoryAllocateInfo allocInfo = {};
	allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
	allocInfo.allocationSize = size;
	allocInfo.memoryTypeIndex = memoryType;
	if (vkAllocateMemory(logicalDevice, &allocInfo, nullptr, &memory) != VK_SUCCESS) { // Could not allocate memory (not enough VRAM or not enough remaining allocations, for example)
		exit(2);
	}

	// Create the first block, taking the chunk's entire space
	Block* block = new Block();
	block->offset = 0; // No offset, starts at 0
	block->size = size; // Takes the chunk's entire space
	block->inUse = false; // Not in use
	block->prev = nullptr;
	block->next = nullptr;
	block->allocationId = 0;
	
	head = block; // Sets the head of the chunk to be the first block
}
```

The **allocate function in the Chunks** is where we find a room for the resource we want to allocate. This function allows us to find a block that is not in use and with enough memory to put the new resource. We can then subdivide this block by creating a new block and remaking the links between the blocks. It is important to understand that we cannot just put a resource right after another in memory, as there are some restrictions on which byte a buffer or an image can be bound on, as described in the [VkMemoryRequirements](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkMemoryRequirements.html) function, in the **alignment** field.
```CPP
VkDeviceSize Chunk::allocate(VkMemoryRequirements memRequirements, VkDeviceSize* allocationNumber, MemoryInfo* memoryInfo) {
	Block* curr = head;
	while (curr) { // While we haven't reached the end of the list
		if (!curr->inUse) { // If the block is not in use
			VkDeviceSize actualSize = curr->size;
			
			// Size of the block minus the alignement (wasted bytes)
			if (curr->offset % memRequirements.alignment != 0) {
				actualSize -= memRequirements.alignment - curr->offset % memRequirements.alignment; // The size of the block minus the wasted bytes
			}

			// If we have enough space to allocate here
			if (actualSize >= memRequirements.size) {
				// Set the size of the block to be the size with alignment
				curr->size = actualSize;

				// Offset with alignment
				if (curr->offset % memRequirements.alignment != 0) {
					curr->offset += memRequirements.alignment - curr->offset % memRequirements.alignment;
				}

				// If we have exactly enough space, there is no need to subdivide
				if (curr->size == memRequirements.size) {
					curr->inUse = true; // The block is now in use
					curr->allocationId = (*allocationNumber)++; // Get the allocation id and increment the number of allocations
					// Fill the memory info structure for deallocation
					memoryInfo->chunkId = id;
					memoryInfo->offset = curr->offset;
					memoryInfo->allocationId = (*allocationNumber) - 1; // The number of allocations has been incremented before so it needs to the allocation id must be reduced by 1 (or just use curr->allocationId)

					return curr->offset;
				}

				// Subdivide the block by creating a new one
				// This block will not be in use at the start, the current one will be in use
				Block* block = new Block();
				newBlock->inUse = false;
				newBlock->offset = curr->offset + memRequirements.size; // The offset of the new block is the offset of the current block plus the size of the resource
				newBlock->size = curr->size - memRequirements.size // The offset of the new block is the size of the current block plus the size of the resource
				newBlock->prev = curr; // The new block is placed after the current block so the previous block of this new block is the current block
				newBlock->next = curr->next; // The next block of this new block is the next block of the current block
				newBlock->allocationId = 0;

				// The current block has been subdivided so it must be relinked and its size changes
				curr->size = memRequirements.size; // The size of the current block is now the size of the data
				curr->inUse = true; // The block is now used by the resource
				if (curr->next) { // If the current block is not the tail
					curr->next->prev = newBlock; // The previous block of the next block of the current block is the new block
				}
				curr->next = newBlock; // The new block is placed after the current block
				curr->allocationId = (*allocationNumber)++; // Get the allocation id and increment the number of allocations

				// Fill the memory info structure for deallocation
				memoryInfo->chunkId = id;
				memoryInfo->offset = curr->offset;
				memoryInfo->allocationId = (*allocationNumber) - 1; // The number of allocations has been incremented before so it needs to the allocation id must be reduced by 1 (or just use curr->allocationId)

				return curr->offset;
			}
		}
		curr = curr->next; // Go to the next block
	}

	return -1;
}
```

The freeBlocks function is used to free all blocks. When creating a new block, we use the **new** expression to allocate the memory for the structure. When using **new**, we must use **delete** to free this memory.
```CPP
	Block* curr;
	while (head) { // While the block list is not empty
		curr = head; // The current block is the head of the list
		head = head->next; // The head is now the block following the head
		delete curr; // Delete the current block
	}
```

### Usage
We now have a simple Vulkan Memory Allocator allowing us to allocate memory for a resource and deallocate this memory when we don't need the resource anymore, but how do we use it ?
For a memory allocator object:
```CPP
MemoryAllocator memoryAllocator;
```
If we want to allocate memory for an image with the **VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT** property, we can do:
```CPP
MemoryInfo memoryInfo = {};
memoryAllocator.allocate(image, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, &memoryInfo);
```
If we don't need this resource anymore, we can do:
```CPP
memoryAllocator.deallocate(memoryInfo.chunkId, memoryInfo.allocationId);
```
This deallocation method requires a way to keep track of the id of the chunk and the id of the allocation, which is why it uses a MemoryInfo structure. It's possible to make a wrapper class Image or Buffer, containing a VkImage or a VkBuffer and a MemoryInfo structure. It's also possible to modify this memory allocator to not use a MemoryInfo structure at all.
# Resources
Good resources about memory allocators:

- **[Vulkan's specification](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#memory)**
- **[AMD - VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)**
- **[Nvidia - Vulkan Memory Management](https://developer.nvidia.com/vulkan-memory-management)**
- **[Kyle Hallyday - A Simple Device Memory Allocator For Vulkan](http://kylehalladay.com/blog/tutorial/2017/12/13/Custom-Allocators-Vulkan.html)**
- **[CPP-Rendering - Vulkan Memory Management : How to write your own allocator](https://cpp-rendering.io/vulkan-memory-management-2/)**
