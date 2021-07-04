# Making Subpass Dependencies
When working on NeigeEngine's synchronization, I had to work on subpass dependencies for the render passes' attachments. Using subpass dependencies allows me to use at least **[vkCmdPipelineBarrier()](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdPipelineBarrier.html)** as possible but it took me some time to understand how to make them. In NeigeEngine, I use multiple render passes with one subpass each for clarity reasons but this should also apply to the case where you have one render pass with multiple subpasses.
## Finding hazards
To fix synchronization issues, the first step is obviously to find them. Some of them are obvious, as they show some graphical artifacts, but some of them won't show up, especially with good computers, capable of running things fast enough to overcome the lack of a proper synchronization.
The easiest way to detect synchronization hazards is to use the **[VK_LAYER_KHRONOS_synchronization2](https://vulkan.lunarg.com/doc/view/1.2.170.0/windows/synchronization2_layer.html)** layer, which can be activated from **Vulkan Configurator** (vkconfig), a software shipped with the Vulkan SDK.
**Note:** It's not recommended to let this layer activated all the time as it destroys performance.
![VK_LAYER_KHRONOS_synchronization2 from Vulkan Configurator](https://i.imgur.com/iOYLxR4.png)
<sub>Activating VK_LAYER_KHRONOS_synchronization2 from Vulkan Configurator</sub>

This layer will show messages about different types of synchronization errors, even some unrelated to render passes synchronization like compute passes synchronization, etc..
**Note:** The layer only shows a limited number of messages. You may also see the same message multiple times.
### Understanding the messages
![Example of a validation message](https://i.imgur.com/UOJRDg8.png)<sub>A Write-after-write hazard on render pass 0x1a8aa6cf148</sub>

The interesting messages for this article are the ones with **type = VK_OBJECT_TYPE_RENDER_PASS**. Those are the ones that concern render passes. **handle = 0xhexcode** is the handle of the render pass presenting an synchronization issue. To find the right render pass, you can either use a debugger to try and find the render pass with the same hexadecimal value or use **[VK_EXT_debug_utils](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VK_EXT_debug_utils.html)** to name your Vulkan objects in validation messages. 
This specific message says that for render pass 0x1a8aa6cf148, in the first subpass (subpass 0) and for the first attachment, which is a color attachment, the layout transition (from initialLayout to finalLayout) specified in the **[VkAttachmentDescription](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkAttachmentDescription.html)** structure for this specific attachment happens before the clear (loadOp), also specified in the same structure. Layout transitions and clears are write operations and must be executed in the right order (clear first, then transition the layout).
## Fixing the subpass dependencies
![The VkSubpassDependency structure](https://i.imgur.com/LmxN4t8.png)
<sub>The [VkSubpassDependency](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkSubpassDependency.html) structure</sub>

The goals of fixing subpass dependencies are:

- Making sure the attachment is not being written or read on when you write on it in the current pass
- Making sure the resources used in the descriptor sets are not being written on when you read them
- Making the layout transition happen at the right time

To do so, we are going to need (at least) two VkSubpassDependency structures. One with srcSubpass as VK_SUBPASS_EXTERNAL and dstSubpass as 0, and one with srcSubpass as 0 and dstSubpass as VK_SUBPASS_EXTERNAL. The numbered subpasses are the render passes' subpasses and VK_SUBPASS_EXTERNAL is a subpass from another render pass.
**Note:** If not specified explicitely by the programmer, both these subpass dependencies are replaced by implicit ones. Those are general ones and we are going to override them for clarity.
![Implicit subpass dependencies](https://i.imgur.com/yXqvt0f.png)<sub>The implicit subpass dependencies (Vulkan specification)</sub>

For the **VK_SUBPASS_EXTERNAL -> 0** dependency:

- srcSubpass = VK_SUBPASS_EXTERNAL
- dstSubpass = 0
- srcStageMask = In which pipeline stages were these attachments used last
- dstStageMask = In which pipeline stages are these attachments going to be used in this pass
- srcAccessMask = What where these attachments' last uses
- dstAccessMask = What are you going to do with these attachments in this pass
- dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT

For the **0 -> VK_SUBPASS_EXTERNAL** dependency:

- srcSubpass = 0
- dstSubpass = VK_SUBPASS_EXTERNAL
- srcStageMask = In which pipeline stages are these attachments going to be used in this pass
- dstStageMask = In which pipeline stages are these attachments going to be used in a later pass
- srcAccessMask = What are you going to do with these attachments in this pass
- dstAccessMask = What are you going to do with these attachments in a later pass
- dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT

To fill these structures, it's good to have a complete overview of what's going on when rendering a frame by writing down all the passes, what the attachments are and what they read from (sampled images in a descriptor set, for example). It's also really important to remember that making frames is a **cycle**.
### Pipeline stage and access mask compatibility
If you put a pipeline stage in **srcStageMask**, then the access in **srcAccessMask** must be compatible with it. This compatibility chart comes from **[Syncmaster 3000](http://s9w.io/syncmaster/)**:
![Pipeline stage and access compatibility chart](https://i.imgur.com/WTdLXuv.png)
<sub>Pipeline stage and access compatibility chart, green means compatible, red means not compatible</sub>
### Example
Let's take a practical example:

**During the making of a frame:**
1. **Depth prepass** (write on a depth image for future usage)
- Attachment: depthPrepassImage (write)
- Read from: nothing
2. **Scene rendering** (write on a color image and read from the depth prepass' image)
- Attachments: sceneColorImage (write), depthPrepassImage (read / write)
- Read from: nothing
3. **Pure depth Screen-Space Ambient Occlusion** (oversimplified, an SSAO method using only a depth buffer)
- Attachment: ssaoColorImage (write)
- Read from: depthPrepassImage (fragment shader)
4. **Post-process** (combine the scene image and the ssao to the swapchain image)
- Attachment: swapchainImage (write)
- Read from : sceneColorImage (fragment shader), ssaoColorImage (fragment shader)
 
With that overview, it's really easy to understand where and how resources are used. We can now write the subpass dependencies:
#### Depth prepass
For the first subpass dependency:
- srcSubpass = **VK_SUBPASS_EXTERNAL**
- dstSubpass = **0**
- srcStageMask = **VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT**, depthPrepassImage was last used in the **previous frame**, read from the **Pure depth SSAO pass** in a **fragment shader**.
- dstStageMask = **VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT | VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT**, for **depth writing**, we want the **early depth test and late depth test stages**.
- srcAccessMask = **0**, or **VK_ACCESS_SHADER_READ_BIT** but this one doesn't need to be specified if srcStageMask is VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT.
- dstAccessMask = **VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT**, we want to **write on the image**, and it's a **depth attachment**.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**

For the second subpass dependency:
- srcSubpass = **0**
- dstSubpass = **VK_SUBPASS_EXTERNAL**
- srcStageMask = **VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT  | VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT**, it is going to be **written on** during **early and late depth tests**.
- dstStageMask = **VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT | VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT**, the depthPrepassImage is going to be used next as a **depth attachment** in the **Scene Rendering pass** to cull occluded fragments.
- srcAccessMask = **VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT**, we are **writing** on it during this render pass and it is a **depth attachment**.
- dstAccessMask = **VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT**, in the **SceneRendering pass**, we are going to **read** from it to cull occluded fragments but we are also going to **store the image**, which counts as a **write** operation.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**
 
#### Scene rendering
When there are multiple attachments in a single render pass, I like to do a pair of subpass dependencies for each of them, which is actually the same as using a bitwise or (**|**) between the flags. For clarity, we are splitting the subpass dependencies here, starting with the **color attachment**:
For the first subpass dependency:
- srcSubpass = **VK_SUBPASS_EXTERNAL**
- dstSubpass = **0**
- srcStageMask = **VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT**, we **read** it in the previous frame in the **Post-process' fragment shader**.
- dstStageMask = **VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT**, we are **writing** on it as a **color attachment**.
- srcAccessMask = **0**, or **VK_ACCESS_SHADER_READ_BIT** but this one doesn't need to be specified if srcStageMask is VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT.
- dstAccessMask = **VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT**, we are **writing** on it, and it is a **color attachment**.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**

For the second subpass dependency:
- srcSubpass = **0**
- dstSubpass = **VK_SUBPASS_EXTERNAL**
- srcStageMask = **VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT**,  we are **writing** on it as a **color attachment**.
- dstStageMask = **VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT**, we are **reading** from it in a **fragment shader** in the **Post-process pass**.
- srcAccessMask = **VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT**, we are **writing** on it during this render pass, and it is a **color attachment**.
- dstAccessMask = **VK_SHADER_READ_BIT**, we are going to **read** from it in a shader in the **Post-process pass**.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**

And for the **depth attachment** :
For the first subpass dependency:
- srcSubpass = **VK_SUBPASS_EXTERNAL**
- dstSubpass = **0**
- srcStageMask = **VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT**, it was last used in a **late depth test** in the **Depth Prepass pass**.
- dstStageMask = **VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT | VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT**, we are **reading** from it in **early and late depth tests** to cull occluded fragments.
- srcAccessMask = **VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT**, it was **written** on during the **depth prepass pass**.
- dstAccessMask = **VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT**, we want to **read** on this image, but also **keep the values** with VK_ATTACHMENT_STORE_OP_STORE, which counts as a **write** operation.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**

For the second subpass dependency:
- srcSubpass = **0**
- dstSubpass = **VK_SUBPASS_EXTERNAL**
- srcStageMask = **VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT |  VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT**, we are **reading** from it in early and late depth tests to cull occluded fragments.
- dstStageMask = **VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT**, we are **reading** from it in a **fragment shader** in the **Pure Depth SSAO pass**.
- srcAccessMask = **VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT**, we are **writing** on it to **keep the values** with VK_ATTACHMENT_STORE_OP_STORE.
- dstAccessMask = **VK_SHADER_READ_BIT**, we are going to **read** from it in a shader in the **Pure Depth SSAO pass**.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**
 
#### Pure depth Screen-Space Ambient Occlusion
For the first subpass dependency:
- srcSubpass = **VK_SUBPASS_EXTERNAL**
- dstSubpass = **0**
- srcStageMask = **VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT**, the ssaoColorImage is **read** in the previous frame in the **Post-process' fragment shader**.
- dstStageMask = **VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT**, we are **writing** on it as a **color attachment**.
- srcAccessMask = **0**, or **VK_ACCESS_SHADER_READ_BIT** but this one doesn't need to be specified if srcStageMask is VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT.
- dstAccessMask = **VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT**, we are **writing** on the image, and it is a **color attachment**.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**

For the second subpass dependency:
- srcSubpass = **0**
- dstSubpass = **VK_SUBPASS_EXTERNAL**
- srcStageMask = **VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT**, we are **writing** on it as a **color attachment**.
- dstStageMask = **VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT**, we will **read** from it in a **fragment shader** in the **Post-process pass**.
- srcAccessMask = **VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT**, we are **writing** on it during this render pass, and it is a **color attachment**.
- dstAccessMask = **VK_SHADER_READ_BIT**, we are going to **read** from it in a shader in the **Post-process pass**.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**

#### Post-process
The post-process pass is quite special, as it's the last render pass and it is writing directly on the swapchain, so don't forget the layout transition **VK_IMAGE_LAYOUT_UNDEFINED -> VK_IMAGE_LAYOUT_PRESENT_SRC_KHR**.

For the first subpass dependency:
- srcSubpass = **VK_SUBPASS_EXTERNAL**
- dstSubpass = **0**
- srcStageMask = **VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT**, the swapchain image was used for **presentation**.
- dstStageMask = **VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT**, we are going to **write** on it as a **color attachment**.
- srcAccessMask = **VK_ACCESS_MEMORY_READ_BIT**, the swapchain image was read for **presentation**.
- dstAccessMask = **VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT**, we are **writing** on the image, and it is a **color attachment**, we are also **transitioning its layout**, which counts as a **write**.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**

For the second subpass dependency:
- srcSubpass = **0**
- dstSubpass = **VK_SUBPASS_EXTERNAL**
- srcStageMask = **VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT**, we are **writing** on it as a **color attachment**.
- dstStageMask = **VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT**, the swapchain image is going to be **presented**.
- srcAccessMask = **VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT**, we are **writing** on it during this render pass, and it is a **color attachment**.
- dstAccessMask = **VK_ACCESS_MEMORY_READ_BIT**, the swapchain image is going to be **presented**.
- dependencyFlags = **VK_DEPENDENCY_BY_REGION_BIT**
 
# Resources
Other good resources about synchronization that I used when I was working on NeigeEngine's sync:

- **[Vulkan's specification](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#synchronization)**
- **[Themaister - Yet another blog explaining Vulkan synchronization](https://themaister.net/blog/2019/08/14/yet-another-blog-explaining-vulkan-synchronization/)**
- **[s9w - Syncmaster 3000](http://s9w.io/syncmaster/)**

