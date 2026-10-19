# The Light View-Projection

A directional light has direction but no position, so use an orthographic light camera fitted around the visible scene. A spot light has a position and cone, so use a perspective projection. Both must use Vulkan’s 0..1 depth convention.

```cpp
glm::mat4 lightProj = glm::ortho(-10.0f, 10.0f, -10.0f, 10.0f,
                                  nearPlane, farPlane);
lightProj[1][1] *= -1.0f;
glm::mat4 lightView = glm::lookAt(sceneCenter - lightDir * distance,
                                  sceneCenter, glm::vec3(0,1,0));
glm::mat4 lightViewProjection = lightProj * lightView;
```

Fit near/far and the orthographic box tightly enough to preserve texel density, but freely refitting every frame can shimmer. The same Y-flip and front-face considerations apply to the shadow pass when it shares culling conventions with the main pass.

A 2048² map over 100 m gives about 4.9 cm per texel; over 400 m it gives 19.5 cm. Coverage and resolution are inseparable.
