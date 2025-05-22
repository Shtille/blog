---
layout: post
title: "Vertex buffer object capabilities"
author: "Shtille"
categories: journal
tags: [thoughts,C,OpenGL]
---

I was trying to find out what is the limit of vertex buffer object allocation size.
So I wrote the program that does such test:

```c
// test if we can allocate buffer
// ------------------------------
bool alloc_buffer(GLsizeiptr size)
{
    GLuint id, error;
    bool result;
    (void)glGetError();
    glGenBuffers(1, &id);
    glBindBuffer(GL_ARRAY_BUFFER, id);
    glBufferData(GL_ARRAY_BUFFER, size, NULL, GL_STATIC_DRAW);
    result = GL_NO_ERROR == glGetError();
    glDeleteBuffers(1, &id);
    return result;
}
// collect capability for a single buffer
// --------------------------------------
GLsizeiptr find_single_buffer_caps()
{
    const GLsizeiptr mb = 1000L * 1000L;
    GLsizeiptr size_step = 1000L * mb; // 1 Gb
    GLsizeiptr size = size_step;
    while (size_step != 0)
    {
        while (alloc_buffer(size))
        {
            size += size_step;
        }
        size -= size_step;
        size_step /= 10;
    }
    return size;
}
```

We just did buffer allocation. I discovered that on many systems results depend on drivers.
For example, on Windows you couldn't even allocate a single gigabyte, on Ubuntu it was like 4 gigabytes and finally on Mac OS X that value would be like 17 terabytes.

Next I tried to pass real buffer to buffer data:

```c
// test if we can allocate buffer
// ------------------------------
bool alloc_buffer(GLsizeiptr size)
{
    GLuint id, error;
    bool result;

    // Allocate real buffer to be tested
    void* data = malloc(size); // 1 Gb
    if (!data)
    {
        fprintf(stdout, "Failed to allocate %zu\n", size);
        return false;
    }
    memset(data, 0, size);

    fprintf(stdout, "alloc buffer of size: %zi", size);

    (void)glGetError();
    glGenBuffers(1, &id);
    glBindBuffer(GL_ARRAY_BUFFER, id);
    glBufferData(GL_ARRAY_BUFFER, size, data, GL_STATIC_DRAW);
    result = GL_NO_ERROR == glGetError();
    glDeleteBuffers(1, &id);

    fprintf(stdout, " %s\n", result ? "succeed" : "failed");

    free(data);

    return result;
}
```

That went fine. Even if we were out of physical memory, system were using swap file.
So OpenGL doesn't provide instruments to obtain physical video memory capabilitites.