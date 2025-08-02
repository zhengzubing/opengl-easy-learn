# 使用 glEGLImageTargetTexture2DOES 替代 glTexImage2D

`glEGLImageTargetTexture2DOES` 是 OpenGL ES 的一个扩展函数，用于将 EGLImage 对象绑定到纹理目标，这比传统的 `glTexImage2D` 方法更高效，特别是在共享图像数据或避免数据拷贝的场景中。

## 主要区别

1. **数据来源**:
   - `glTexImage2D`: 从内存中的像素数据上传到纹理
   - `glEGLImageTargetTexture2DOES`: 直接使用现有的 EGLImage 作为纹理数据源

2. **性能优势**:
   - 避免数据拷贝(当 eglCreateImageKHR 实现真正的免拷贝时，其本质是通过 内存共享 而非硬件拷贝实现的)
   - 支持跨进程/跨API图像共享
   - 适合视频纹理等频繁更新的场景

## 使用方法

```c
// 1. 首先创建EGLImage (通常来自相机、视频解码器等)
EGLImageKHR image = eglCreateImageKHR(...);

// 2. 绑定纹理
glBindTexture(GL_TEXTURE_2D, textureId);

// 3. 使用EGLImage作为纹理数据源
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, image);

// 4. 使用完后可以释放EGLImage (纹理会保留内容)
eglDestroyImageKHR(image);
```


eglCreateImageKHR 详解
eglCreateImageKHR 是 EGL 的一个扩展函数，用于创建 EGLImage 对象（EGLImage 本身并不直接"拥有"内存，而是一个跨API的图像数据引用机制，它指向的内存位置取决于创建时使用的 target 和 buffer 参数。），这是一种跨 API 共享图像数据的机制。它是 EGL_KHR_image 或 EGL_KHR_image_base 扩展的一部分。

函数原型
```c
EGLImageKHR eglCreateImageKHR(
    EGLDisplay dpy,
    EGLContext ctx,
    EGLenum target,
    EGLClientBuffer buffer,
    const EGLint* attrib_list);
// 参数说明
// dpy: EGL 显示连接
// ctx: 创建图像的上下文 (可为 EGL_NO_CONTEXT)
// target: 指定图像源类型，常见值：
//  EGL_GL_TEXTURE_2D_KHR: 从 OpenGL ES 2D 纹理创建
//  EGL_NATIVE_BUFFER_ANDROID: Android 原生缓冲区
//  EGL_GL_RENDERBUFFER_KHR: 从 OpenGL ES 渲染缓冲区创建
//  EGL_LINUX_DMA_BUF_EXT: 通过 Linux DMA-BUF 机制共享内存缓冲区
// buffer: 与 target 对应的缓冲区句柄
// attrib_list: 属性列表，以 EGL_NONE 结尾


// demo:
const EGLint dma_buf_attribs[] = {
    EGL_LINUX_DRM_FOURCC_EXT, DRM_FORMAT_ARGB8888,  // 指定DRM格式
    EGL_WIDTH,  1920,                               // 图像宽度
    EGL_HEIGHT, 1080,                               // 图像高度
    EGL_DMA_BUF_PLANE0_FD_EXT, dmabuf_fd,           // DMA-BUF文件描述符 // 提前分配好的DMA buffer
    EGL_DMA_BUF_PLANE0_OFFSET_EXT, 0,               // 数据偏移量
    EGL_DMA_BUF_PLANE0_PITCH_EXT, 7680,             // 行跨度(字节)  // 必须满足要求的内存对齐（如 64 字节对齐）, 否则eglCreateImageKHR会失败
    EGL_NONE  // 必须的结束标记
};

EGLImageKHR image = eglCreateImageKHR(
    display,
    EGL_NO_CONTEXT,
    EGL_LINUX_DMA_BUF_EXT,
    NULL,
    dma_buf_attribs);
```
