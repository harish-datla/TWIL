# Descriptor Sets

A descriptor connects a shader slot to a resource. For a combined image sampler, create a set layout, a descriptor pool, an allocated set, and a write that supplies the image view, sampler, and layout. Bind the set against the pipeline layout at draw time.

```cpp
VkDescriptorSetLayoutBinding binding{};
binding.binding = 1;
binding.descriptorCount = 1;
binding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
binding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;
VkDescriptorSetLayoutCreateInfo layoutInfo{
    VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO };
layoutInfo.bindingCount = 1; layoutInfo.pBindings = &binding;
vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &setLayout);

VkDescriptorPoolSize size{
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, MAX_FRAMES };
VkDescriptorPoolCreateInfo poolInfo{
    VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO };
poolInfo.maxSets = MAX_FRAMES; poolInfo.poolSizeCount = 1;
poolInfo.pPoolSizes = &size;
vkCreateDescriptorPool(device, &poolInfo, nullptr, &pool);

VkDescriptorImageInfo imageInfo{};
imageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
imageInfo.imageView = textureImageView; imageInfo.sampler = textureSampler;
VkWriteDescriptorSet write{ VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET };
write.dstSet = set; write.dstBinding = 1; write.descriptorCount = 1;
write.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
write.pImageInfo = &imageInfo;
vkUpdateDescriptorSets(device, 1, &write, 0, nullptr);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout, 0, 1, &set, 0, nullptr);
```

Pool capacity is required both per descriptor type and in total sets. One set per frame in flight for 100 materials with three textures each needs 200 sets and 600 combined image samplers. Validation may not catch an undersized pool consistently. Do not update a set the GPU is using; update before submission, use one set per frame, or opt into update-after-bind. Separate sampled-image and sampler descriptors can reduce duplicated data but require more bookkeeping.
