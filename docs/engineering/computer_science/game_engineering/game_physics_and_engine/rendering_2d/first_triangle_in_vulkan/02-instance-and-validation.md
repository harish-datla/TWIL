# Instance and Validation

`VkInstance` is the application’s connection to Vulkan. Provide application information, the API version, platform surface extensions, and the validation layer. Wire `VK_EXT_debug_utils` to send validation messages to the log.

`VK_API_VERSION_1_3` is `(1 << 22) | (3 << 12) = 4,206,592`. It states the version targeted; it does not enable device features. The surface is an instance extension, while the swapchain is a device extension.

Enable `VK_LAYER_KHRONOS_validation` in debug builds only after checking `vkEnumerateInstanceLayerProperties`; otherwise instance creation fails with `VK_ERROR_LAYER_NOT_PRESENT`. A bootstrap debug messenger in `pNext` catches errors during instance creation itself.

```cpp
VkApplicationInfo appInfo{ VK_STRUCTURE_TYPE_APPLICATION_INFO };
appInfo.apiVersion = VK_API_VERSION_1_3;
const char* layers[] = { "VK_LAYER_KHRONOS_validation" };
VkInstanceCreateInfo info{ VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO };
info.pApplicationInfo = &appInfo;
info.enabledLayerCount = 1;
info.ppEnabledLayerNames = layers;
info.enabledExtensionCount = uint32_t(extensionNames.size());
info.ppEnabledExtensionNames = extensionNames.data();
VkResult result = vkCreateInstance(&info, nullptr, &instance);
```
