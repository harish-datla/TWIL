# Devices and Queues

Enumerate physical devices, score them, and choose one that supports the swapchain extension. Then find a graphics queue family and a present family. Graphics support is a capability bit; present support is a surface-specific query, so the families may be the same or different.

Deduplicate family indices before creating queues. If graphics and present differ, create both queues and choose concurrent swapchain sharing or explicit ownership transfers. A feature must be enabled, not merely reported as available; for Vulkan 1.3, dynamic rendering is enabled through `VkPhysicalDeviceVulkan13Features`.

```cpp
VkBool32 presentSupport = VK_FALSE;
vkGetPhysicalDeviceSurfaceSupportKHR(physicalDevice, familyIndex,
                                     surface, &presentSupport);
std::set<uint32_t> uniqueFamilies = { graphicsFamily, presentFamily };
VkPhysicalDeviceVulkan13Features features13{
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_3_FEATURES };
features13.dynamicRendering = VK_TRUE;
const char* extensions[] = { VK_KHR_SWAPCHAIN_EXTENSION_NAME };
// Build one VkDeviceQueueCreateInfo per unique family, then chain
// features13 into VkDeviceCreateInfo::pNext and call vkCreateDevice.
```
