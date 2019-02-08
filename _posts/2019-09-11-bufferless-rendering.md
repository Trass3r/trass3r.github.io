---
title: Bufferless rendering in OpenGL
categories: coding
tags: cpp, opengl
---

A simple operation like rendering a quad to run a fullscreen pixel shader pass requires a lot of boilerplate setup code, something like:

```cpp
const GLfloat vertices[] = {
	// Positions | Texture Coords
	 1.0f,  1.0f, 1.0f, 1.0f,
	 1.0f, -1.0f, 1.0f, 0.0f,
	-1.0f, -1.0f, 0.0f, 0.0f,
	-1.0f,  1.0f, 0.0f, 1.0f
};
const GLuint indices[] = {
	0, 1, 3,
	1, 2, 3
};
glGenVertexArrays(1, &VAO);
glBindVertexArray(VAO);
glGenBuffers(1, &VBO);
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

glGenBuffers(1, &EBO);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 4 * sizeof(GLfloat), (GLvoid*)0);
glEnableVertexAttribArray(0);
glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 4 * sizeof(GLfloat), (GLvoid*)(2 * sizeof(GLfloat)));
glEnableVertexAttribArray(1);
```

This becomes even more pronounced in lower-level APIs like Vulkan.

## A better way

But in a special case like this two "optimizations" can be done.

1. Draw a triangle covering the full screen instead of a quad. It may still be split up into smaller triangles by the hardware but it's less code for us.
2. Use bufferless rendering.

When rendering without any buffers the vertex shader will simply be invoked the number of specified times without input data:
```cpp
glDrawArrays(GL_TRIANGLES, 0, 3);
```

So instead of receiving your vertices as usual:
```glsl
layout (location = 0) in vec2 position;
layout (location = 1) in vec2 texCoord;
```
you can simply move the constant vertices array to the shader or even compute it on the fly like this:
```glsl
vec2 position = vec2(gl_VertexID % 2, gl_VertexID / 2) * 4.0 - 1;
vec2 texCoord = (position + 1) * 0.5;
```

![fullscreen triangle illustration](/assets/fullscreentriangle.svg)

We abuse `gl_VertexID` to generate a triangle with vertices (-1, -1), (3, -1), (-1, 3) and tex coords (0, 0), (2, 0), (0, 2).

Here is the final vertex shader and the SPIR-V code it generates: <http://shader-playground.timjones.io/89f6836c1f678a100f9bddfa07d866cf>

## Final remarks

There is just one little snag. You still need a VAO when using the core profile because:

> A non-zero Vertex Array Object must be bound (though no arrays have to be enabled, so it can be a freshly-created vertex array object).
[<sup>1</sup>](https://www.khronos.org/opengl/wiki/Vertex_Rendering#Prerequisites)

> The compatibility OpenGL profile makes VAO object 0 a default object. The core OpenGL profile makes VAO object 0 not an object at all.
[<sup>2</sup>](https://www.khronos.org/opengl/wiki/Vertex_Specification#Vertex_Array_Object)

So just use a dummy VAO.