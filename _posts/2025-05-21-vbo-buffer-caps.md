---
layout: post
title: "Vertex buffer object capabilities"
author: "Shtille"
categories: journal
tags: [thoughts,C,OpenGL]
---

## Overview

I was trying to find out what is the limit of vertex buffer object allocation size in OpenGL.
So I wrote the program that does such test.

## Process

### Buffer allocation with no data

At first I tried just pure buffer allocation without any data with `glBufferData` function:

```c
// test if we can allocate buffer
// ------------------------------
bool test_buffer_allocation(GLsizeiptr size)
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
        while (test_buffer_allocation(size))
        {
            size += size_step;
        }
        size -= size_step;
        size_step /= 10;
        if (size == 0)
            size = size_step;
    }
    return size;
}
```

I discovered that on many systems results depend on drivers. For example, on Windows you couldn't even allocate a single gigabyte, on Ubuntu it was like 4 gigabytes and finally on Mac OS X that value would be like 17 terabytes.

### Buffer allocation with a data

Next I tried to pass real buffer to buffer data:

```c
// test if we can allocate buffer
// ------------------------------
bool test_buffer_allocation(GLsizeiptr size)
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

I got successful allocation results like 20 gigabytes buffer while on test system I had only discrete videocard with 8 Gb (GeForce RTX 4070). So system were using swap file during these allocations. That isn't what I wanted.

### Buffer allocation with data comparison

I decided to use `glMapBuffer` to retrieve actual video memory data, and compare it afterwards.

```c
bool test_buffer_allocation(GLsizeiptr size)
{
    GLuint id, error;
    bool result;

    // Allocate real buffer to be tested
    void* data = malloc(size); // 1 Gb
    if (!data)
    {
        fprintf(stdout, "Failed to allocate %zu\n bytes with malloc", size);
        return false;
    }
    const GLsizeiptr block_size = 1000;
    unsigned char* initial_data = (unsigned char*) data;
    unsigned int n = 0;
    for (GLsizeiptr i = 0; i < size; i += block_size)
    {
        unsigned char c = (unsigned char)(n++ % 256);
        if (i + block_size <= size)
            memset(initial_data + i, c, block_size);
        else
            memset(initial_data + i, c, size - i);
    }

    fprintf(stdout, "alloc buffer of size: %zi", size);

    (void)glGetError();
    glGenBuffers(1, &id);
    glBindBuffer(GL_ARRAY_BUFFER, id);
    glBufferData(GL_ARRAY_BUFFER, size, data, GL_STATIC_DRAW);
    result = GL_NO_ERROR == glGetError();

    fprintf(stdout, " %s\n", result ? "succeed" : "failed");

    // Compare data
    if (result)
    {
        const unsigned char* mapped_data = (const unsigned char*) glMapBuffer(GL_ARRAY_BUFFER, GL_READ_ONLY);
        if (GL_NO_ERROR != glGetError())
        {
            fprintf(stdout, "Failed to map buffer\n");
            glDeleteBuffers(1, &id);
            free(data);
            return false;
        }
        
        bool same = memcmp(initial_data, mapped_data, size) == 0;
        glUnmapBuffer(GL_ARRAY_BUFFER);
        fprintf(stdout, "Buffer data check %s\n", same ? "succeed" : "failed");

        if (!same)
        {
            glDeleteBuffers(1, &id);
            free(data);
            return false;
        }
    }

    glDeleteBuffers(1, &id);

    free(data);

    return result;
}
```

The results I achieved is really the ones I wanted: I got about 4 gigabyte result size (with actual videocard memory size of 8 gigabytes). But the process takes too much time. I need optimizations.

### Optimizations

So I made following optimizations:

* Descrease number of iterations for initial buffer data fill
* Don't use `if` in cycle.
* Compare only the last step of iteration
* Map buffer range instead of entire buffer
* Don't allocate heap memory each iteration

### Problems

1. `glMapBufferRange` with offset out of valid data will cause application crash
    ```c
    glMapBufferRange(GL_ARRAY_BUFFER, offset, mapped_size, GL_MAP_READ_BIT);
    ```
2. `memcmp` with offset out of valid data will cause application crash
    ```c
    memcmp(initial_data + offset, mapped_data + offset, mapped_size) == 0;
    ```

Both crashes are caused by access violation reading location. Hence regular `memcmp` works fine.

### Add progress indicator

We can add console progress indicator to not confuse user:

```c
// updates console progress indicator
// ----------------------------------
void update_indicator()
{
    static const char indicator_array[4] = {'\\', '|', '/', '-'};
    fputc('\b', stdout);
    fputc(indicator_array[counter++ % 4], stdout);
}
```

## The final code

```c
// allocates buffer and fills with test data
// -----------------------------------------
void* allocate_buffer(GLsizeiptr size)
{
    void* data = NULL;
    data = malloc(size);
    if (!data)
        return NULL;
    const GLsizeiptr num_iterations = 1000;
    GLsizeiptr block_size = size / num_iterations;
    if (block_size == 0)
        block_size = 1;
    GLsizeiptr div = size / block_size;
    GLsizeiptr mod = size % block_size;
    GLsizeiptr base_size = block_size * div; // base_size + mod = size
    // Divide buffer fill with data into two parts
    unsigned char* byte_data = (unsigned char*) data;
    unsigned int n = 0;
    for (GLsizeiptr i = 0; i < base_size; i += block_size)
    {
        unsigned char c = (unsigned char)(n++ % 256);
        memset(byte_data + i, c, block_size);
    }
    {
        unsigned char c = (unsigned char)(n++ % 256);
        memset(byte_data + base_size, c, mod);
    }
    return data;
}

// test if we can allocate buffer
// ------------------------------
bool test_buffer_allocation(void* data, GLsizeiptr size, GLsizeiptr offset)
{
    GLuint id;
    bool result;

    fprintf(stdout, "VBO allocation of size: %zi", size);

    (void)glGetError();
    glGenBuffers(1, &id);
    glBindBuffer(GL_ARRAY_BUFFER, id);
    glBufferData(GL_ARRAY_BUFFER, size, data, GL_STATIC_DRAW);
    result = GL_NO_ERROR == glGetError();

    fprintf(stdout, " %s\n", result ? "succeed" : "failed");

    // Compare data
    if (result)
    {
        const unsigned char* initial_data = (const unsigned char*) data;
        static_assert(sizeof(GLsizeiptr) == sizeof(GLintptr), "OpenGL types size mismatch");
        GLsizeiptr mapped_size = size - offset;
        const unsigned char* mapped_data = (const unsigned char*) glMapBuffer(GL_ARRAY_BUFFER, GL_READ_ONLY);
        // const void* mapped_data = glMapBufferRange(GL_ARRAY_BUFFER, offset, mapped_size, GL_MAP_READ_BIT);
        if (GL_NO_ERROR != glGetError())
        {
            fprintf(stdout, "Failed to map buffer\n");
            glDeleteBuffers(1, &id);
            return false;
        }
        
        // bool same = memcmp(initial_data + offset, mapped_data, mapped_size) == 0;
        // bool same = memcmp(initial_data + offset, mapped_data + offset, mapped_size) == 0;
        bool same = memcmp(initial_data, mapped_data, size) == 0;
        glUnmapBuffer(GL_ARRAY_BUFFER);
        fprintf(stdout, "Buffer data check %s\n", same ? "succeed" : "failed");

        if (!same)
            result = false;
    }

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
    GLsizeiptr heap_size = 0;
    void* heap_data = NULL;
    while (size_step != 0)
    {
        bool result = true;
        while (result)
        {
            if (heap_size < size)
            {
                heap_data = allocate_buffer(size);
                if (!heap_data)
                {
                    fprintf(stdout, "Failed to allocate %zu bytes of heap\n", size);
                    return 0;
                }
                heap_size = size;
            }
            result = test_buffer_allocation(heap_data, size, size - size_step);
            if (result)
                size += size_step;
        }
        size -= size_step;
        size_step /= 10;
        if (size == 0)
            size = size_step;
    }
    if (heap_data != NULL)
        free(heap_data);
    return size;
}
```

## Resume

I found a way to determine buffer allocation capability. But it take quite a long time to get the final value.
For my GeForce RTX 4070 with 8 Gb of video memory the available single buffer size is about 4 GB.