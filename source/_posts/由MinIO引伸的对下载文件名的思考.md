---
title: 由MinIO引伸的对下载文件名的思考
date: 2024-03-07 16:05:11
tags: [Java,MinIO,对象储存]
categories: Java
---

# 问题描述

在博主接手的一个项目中，对于文件的储存使用了`MinIO`，文件的上传和下载原本已经都实现了，测试也没有问题。

如图,图片预览：![image-20240307161629672](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2024/03/7_16_28_41_image-20240307161629672.png)但是最近博主想上传一个`Execl`文件作为信息导入模板，上传并没有问题，文件也能下载，但是下载的文件名却十分的奇怪：

![image-20240307163013323](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2024/03/7_16_30_13_image-20240307163013323.png)

和我上传的文件名完全没关系啊，顿时博主一头雾水。

# 问题分析

关于文件下载，是通过向后端发送请求，获取到文件的URL，然后前端通过`window.open()`方法打开连接尝试下载：

```js
// 下载学生信息模板
const downloadTemp = () =>{
  const req = { configKey: 'learnerTemplate'}
  sysConfigList(req).then((res) =>{
    if (res.length > 0){
      const configValue = res[0].configValue
      
      // window.open实现
      window.open(configValue, '_self')
    }
  }).catch((err) =>{
    ElMessage({
      type: 'error',
      message: err
    })
  })
}
```

遇到这个问题的时候博主首先想到的是换成通过`a`标签实现下载，在`a`标签里面可以通过指定`download`属性来指定文件名：

```js
const downloadTemp = () =>{
	const req = { configKey: 'learnerTemplate'}
	sysConfigList(req).then((res) =>{
	if (res.length > 0){
		const configValue = res[0].configValue
        
        // 用a标签实现下载
        const link = document.createElement('a')
        link.style.display = 'none';
        // 设置下载地址
        link.setAttribute('href', row.configValue)
        // 设置文件名
        link.setAttribute('download', row.configName)
        link.setAttribute('target', '_blank')
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    }
    }).catch((err) =>{
        ElMessage({
            type: 'error',
            message: err
        })
    })

}
```

但是用了这个方法下载的文件名还是和原来没有区别。

那先分析一下为啥这种方式会失效呢？直接`chatGPT`一下：

> 1. **文件跨域问题：** 如果链接的文件位于与当前页面不同的域，某些浏览器可能会限制对跨域资源的下载。这是出于安全考虑，以防止跨站点请求伪造（CSRF）等攻击。
> 2. **文件服务器配置问题：** 在某些情况下，文件服务器可能会配置为不允许直接下载文件，或者可能需要某些特定的身份验证或权限才能下载文件。
> 3. **文件路径问题：** 如果链接的文件路径不正确或文件不存在，浏览器可能无法正常下载文件。确保链接的文件路径正确且文件存在于服务器上。
> 4. **浏览器安全设置：** 某些浏览器安全设置可能会阻止对某些文件类型的下载，或者要求用户在下载前确认。用户可以通过调整浏览器的安全设置来解决这个问题。

分析一下我的资源文件URL,明显是`MinIO`服务器和我微服务**不在同一个IP下**，也就是形成了**跨域**，于是 **download属性失效**。

顺便温故一下跨域有哪些情况：

> 1. **不同的协议（Protocol）：** 比如从 `http://` 的页面请求 `https://` 的资源，或者相反，由于安全策略的限制，浏览器不允许这种情况下的跨域请求。
> 2. **不同的域名（Domain）：** 比如从 `example.com` 的页面请求 `anotherdomain.com` 的资源，由于浏览器的同源策略，不同域名之间的 JavaScript 请求是受限制的。
> 3. **不同的子域名（Subdomain）：** 比如从 `sub1.example.com` 的页面请求 `sub2.example.com` 的资源，虽然它们属于同一个根域名 `example.com`，但是浏览器默认情况下也不允许这种跨子域的请求，但可以通过设置 CORS（跨域资源共享）来进行配置。
> 4. **不同的端口号（Port）：** 比如从 `http://example.com:8080` 的页面请求 `http://example.com:3000` 的资源，由于不同的端口号也被视为不同的域，同样受到跨域限制。



既然这种方法不行，那么只能尝试其他的方法。进一步分析，下载的资源的文件名由谁指定的呢？凭借多年下载文件的直觉，我感觉是由资源的URL名字来决定的，比如`http://example.com/导入模板.xlsx`这个文件，他的文件名应该就是最后的：'~~导入模板.xlsx~~'。但是实际上在我的资源`URL`和下载的过来的文件名没有半毛钱关系。显然这种情况和我实际生活经验有出入。

那就只能从后端上传储存的接口来分析了。

首先看看前端上传逻辑是什么：

```js
export function uploadDoc(data, cb) {
  const formData = new FormData()
  formData.append('docFile', data.file)
  const config = {
    onUploadProgress: progressEvent => {
      const videoUploadPercent = Number((progressEvent.loaded / progressEvent.total * 100).toFixed(2))
      // 计算上传进度
      if (cb) {
        cb(videoUploadPercent)
      }
    }
  }
  return request.post('/system/admin/upload/doc', formData, config)
}
```

正常的通过表单上传文件，这个时候我忽然注意到表单数据名`docFile`不正是我下载的文件的文件名吗？那必然是后端哪里的逻辑把下载的文件的文件名直接给指定了。于是进一步看看后端怎么处理的吧。

```java
    /**
     * 上传文件到MiniIO
     *
     * @param storageConfig bucket名称
     * @param fileDir       文件目录
     * @param fileName      文件名称
     * @param stream        文件流
     * @return 上传后的文件地址
     * @throws Exception 上传异常
     */
    private String uploadForMinio(Upload storageConfig, String fileDir, String fileName, String fileOriginalName, String contentType, boolean publicRead, InputStream stream) throws Exception {
        MinioClient minioClient = getMinioClient(storageConfig);

        if (!minioClient.bucketExists(BucketExistsArgs.builder().bucket(storageConfig.getMinioBucket()).build())) {
            // 创建Bucket
            minioClient.makeBucket(MakeBucketArgs.builder().bucket(storageConfig.getMinioBucket()).build());
            // 设置Bucket访问策略
            String policy = ""
            minioClient.setBucketPolicy(
                SetBucketPolicyArgs.builder().bucket(storageConfig.getMinioBucket())
                .config(policy.replace("{bucket}", storageConfig.getMinioBucket()))
                .build());
        }

        // 处理前缀目录
        String objectName = StringUtils.hasText(fileDir) ? fileDir + "/" + fileName : fileName;

        // 设置文件下载名称
        Map<String, String> headerMap = new HashMap<>();
        headerMap.put("Content-Disposition", "attachment;filename=" + java.net.URLEncoder.encode(fileOriginalName, "UTF-8"));
        if (StringUtils.hasText(contentType)) {
            headerMap.put("Content-Type", contentType);
        }

        minioClient.putObject(PutObjectArgs.builder().bucket(storageConfig.getMinioBucket()).object(objectName).headers(headerMap).stream(stream, stream.available(), -1).build());
        return storageConfig.getMinioBucket() + "/" + objectName;
    }
```

还好上个人有点良心写了注释，很快就能知道了后端是通过**设置响应头来指定了文件名**的。也就是上面的：

```java
headerMap.put("Content-Disposition", "attachment;filename=" + java.net.URLEncoder.encode(fileOriginalName, "UTF-8"));
```

再仔细一看，在哪里有调用这个方法，直接查找用法，终于发现问题所在了：

```java
public String uploadPic(MultipartFile file, Upload upload) {
    try {
        String fileName = IdUtil.simpleUUID() + "." + FileUtil.getSuffix(file.getOriginalFilename());
        String filePath = uploadForMinio(upload, "education", fileName, file.getName(), 
                                         file.getContentType(), true, file.getInputStream());
        return getMinioFileUrl(upload.getMinioDomain(), filePath);
    } catch (Exception e) {
        log.error("MinIO上传错误", e);
    }
    return "";
}
```

再看看前端响应是不是这个：

![image-20240307172608463](https://gitlab.com/Echo-xzp/Resource/-/raw/main/img/2024/03/7_17_26_8_image-20240307172608463.png)

相应头正是这个，也就是这就是问题的根源了。

大无语事件，写这个方法的人把参数`fileName`传了个`file.getName()`进去，这里他应该是想传`file.getOriginalFilename()`进去的吧，一时粗心写错了，于是造成了下载的文件名都为表单字段名。~~(蚌埠住了)~~

回忆一下这两个方法的区别：

> 1. **getOriginalFilename()：**
>    - `getOriginalFilename()` 方法返回客户端上传的文件的原始文件名。
>    - 这个原始文件名可能包含路径信息，但并不代表文件在服务器上的实际名称。
>    - 如果客户端上传的文件没有提供原始文件名，或者上传的不是文件，而是一个空的 `MultipartFile` 对象，则该方法可能返回 `null` 或空字符串。
> 2. **getName()：**
>    - `getName()` 方法返回 `MultipartFile` 对象在上传时使用的参数名。
>    - 通常，这个参数名是表单中 `<input>` 元素的 `name` 属性值。
>    - 对于单文件上传，通常情况下 `getName()` 返回的就是上传表单中 `<input type="file">` 元素的 `name` 属性值。
>    - 对于多文件上传，如果多个文件使用了相同的参数名进行上传，则 `getName()` 返回的就是这个参数名。

# 问题解决

那知道问题所在就行了，把那个`getName()`方法改成`getOriginalFilename()`，一下就解决了。

进一步确定一下我的分析，`chatGPT`一下，文件名到底由谁决定：

> 当通过 URL 下载文件时，文件名通常由两个主要因素决定：
>
> 1. **服务器端的响应头：** 通常情况下，服务器会在响应头中包含一个 `Content-Disposition` 头部来指定文件名。这个头部的值通常是 `attachment`，表示告知浏览器要以附件的形式下载资源，并且可以通过 `filename` 参数来指定下载的文件名。例如：
>
>    ```
>    Content-Disposition: attachment; filename="example.txt"
>    ```
>
>    在这个例子中，浏览器将下载名为 `example.txt` 的文件。
>
> 2. **URL 中的路径或参数：** 如果服务器未提供 `Content-Disposition` 头部，或者浏览器不支持该头部，那么浏览器可能会根据 URL 的路径或参数来确定下载的文件名。例如，在以下 URL 中：
>
>    ```
>    https://example.com/download?file=example.txt
>    ```
>
>    浏览器可能会尝试使用 `example.txt` 作为下载文件的默认文件名。
>
> 在实际应用中，通常会优先考虑服务器端的 `Content-Disposition` 头部来指定下载的文件名，因为这样可以更精确地控制下载时的文件名。

那也证明我刚才说的生活经验也没错，两种方法都决定文件名，只不过先后问题。
