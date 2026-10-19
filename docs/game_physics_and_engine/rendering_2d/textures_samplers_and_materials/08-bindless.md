# Bindless

With hundreds of materials, binding a descriptor set for every draw adds CPU work and state changes. Descriptor indexing—often called bindless—binds a large array of texture descriptors once and passes each draw a material index. With 500 materials, the frame may bind once instead of up to 500 times, while the shader samples `textures[materialIndex]`.

Query and enable the relevant optional features: `runtimeDescriptorArray`, `descriptorBindingPartiallyBound`, `descriptorBindingSampledImageUpdateAfterBind`, and `shaderSampledImageArrayNonUniformIndexing`. When indices differ across invocations, use the appropriate `nonuniformEXT(...)` qualifier; otherwise behavior is undefined by the Vulkan specification. Descriptor indexing is distinct from the newer descriptor-buffer extension.
