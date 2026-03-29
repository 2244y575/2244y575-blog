+++
date = '2026-03-29T19:11:47+08:00'
draft = false
title = '一文搞懂阿里oss存储'
+++

## 一、OSS介绍
### 1. OSS是什么
**OSS**又称**对象存储**服务，类似于一个巨大的云端仓库，用来存储和管理文件，供服务端或用户上传和读取。
### 2. 为什么要使用阿里OSS
#### - 直接存储文件
- 传统服务中，我们常常会遇到需要保存文件相关的需求，虽然可以直接将二进制流存入数据库来保存文件，但是这样会极大的增加数据库的存储负担，而且大文件也不适合直接存储。
- 直接把数据存储到单台服务器上，如果出现损坏或者数据丢失，很难找回。
#### - OSS存储
- 使用OSS存储将文件放置到云服务器上，极大的缓解了数据库的压力，数据库只需要存储对应文件的url或osskey，相比原文件大小下降数十倍不止。
- 上传到云服务器后数据由OSS方管理和存储，数据安全性更得到保障。
- OSS还提供多种存储类型，可以对不常访问的数据进行归档等。
## 二、怎么使用阿里OSS服务
### 1. 创建阿里OSS
**1.1 进入Oss页面**

进入阿里云官网，进入控制台，选择对象存储OSS服务。

**1.2 创建Bucket**

在Bucket列表中创建Bucket，根据需求选择配置，这里给出我的配置。

![创建阿里OSS](/images/create_alioss.png)

> **注意：** 创建后，要使用还需要进行对应费用支付，学习阶段建议购买资源包，6个月也只需要不到5块。
### 2. 接入springboot服务
**2.1 引入alioss依赖**

在pom.xml中引入阿里云OSS的SDK依赖。
```xml
<dependency>
    <groupId>com.aliyun.oss</groupId>
    <artifactId>aliyun-sdk-oss</artifactId>
    <version>${aliyun-oss.version}</version>
</dependency>
```
**2.2 配置oss配置文件**

在 `application.yml` 中配置 OSS 的连接信息。为了安全，**千万不要**把这些信息硬编码在 Java 文件里。

- **endpoint**：你的bucket选择的地域对应的endpoint，例如使用华南1即为**oss-cn-shenzhen.aliyuncs.com**

- **bucket-name：**
你创建的bucket名称

- **access-key-id**：在阿里云官网获取
- **access-key-secret**：在阿里云官网获取

![阿里云access key入口](/images/goto_ali_access_key.png)
![创建access key](/images/create_access_key.png)

**2.3 创建alioss工具类**

为了方便使用阿里服务，一般选择创建对应工具类。

oss存储分为私有存储和公有存储
- **公有存储**：用户可以直接访问存储的文件。
- **私有存储**：用户不能访问存储的文件，需要生成对应文件的临时访问url才可以正常访问。

一般使用私有存储较多，保证数据的安全性，而一些常用的公有访问的例如头像文件可以设置公有存储。
```java
public class AliOssUtils {

    private final AliOssProperties aliOssProperties;

    /**
     * 上传文件到公共读目录
     * @param file 上传的文件
     * @param directory OSS 目录路径
     * @return 文件访问URL
     */
    public String uploadToPublicDirectory(MultipartFile file, String directory) {
        return uploadFile(file, directory, true);
    }

    /**
     * 上传文件到私有目录
     * @param file 上传的文件
     * @param directory OSS 目录路径
     * @return 文件对象路径
     */
    public String uploadToPrivateDirectory(MultipartFile file, String directory) {
        return uploadFile(file, directory, false);
    }

    /**
     * 生成私有文件的临时访问URL（带签名）
     * @param objectPath 对象路径 oss://bucket/object
     * @param expirationMinutes 过期时间（分钟）
     * @return 带签名的临时URL
     */
    public String generatePrivateFileUrl(String objectPath, int expirationMinutes) {
        if (objectPath == null || !objectPath.startsWith("oss://")) {
            log.error("对象路径格式不正确，应以oss://开头");
            throw new DataNotFoundException("未找到对应文件");
        }
        // 解析对象路径
        String[] parts = objectPath.replace("oss://", "").split("/", 2);
        String bucketName = parts[0];
//        aliOssProperties.getBucketName()
        String objectName = parts[1];

        OSS ossClient = new OSSClientBuilder().build(aliOssProperties.getEndpoint(), aliOssProperties.getAccessKeyId(), aliOssProperties.getAccessKeySecret());
        try {
            // 生成预签名URL
            java.util.Date expiration = new java.util.Date(
                    new java.util.Date().getTime() + (long) expirationMinutes * 60 * 1000
            );
            return ossClient.generatePresignedUrl(bucketName, objectName, expiration).toString();
        } finally {
            if (ossClient != null) {
                ossClient.shutdown();
            }
        }
    }

    /**
     * 下载部分文件
     * @param url 文件url
     * @param target 本地临时文件路径
     * @param start 起始字节偏移量
     * @param len 要下载的字节长度
     */
    public void downloadRange(String url, Path target, long start, long len) throws IOException {
        HttpURLConnection conn = (HttpURLConnection) new URL(url).openConnection();
        conn.setRequestProperty("Range", "bytes=" + start + "-" + (start + len - 1));
        conn.setConnectTimeout(3000);
        conn.setReadTimeout(8000);
        try (InputStream in = conn.getInputStream()) {
            Files.copy(in, target, StandardCopyOption.REPLACE_EXISTING);
        }
        if (conn.getResponseCode() != HttpURLConnection.HTTP_PARTIAL) {
            throw new IOException("Range 下载失败，code=" + conn.getResponseCode());
        }
    }

    /**
     * 判断oss文件是否存在
     * @param ossKey 文件ossKey
     * @return 文件是否存在
     */
    public boolean checkFileExist(String ossKey) {
        OSS ossClient = new OSSClientBuilder().build(aliOssProperties.getEndpoint(), aliOssProperties.getAccessKeyId(), aliOssProperties.getAccessKeySecret());
        String objectName;
        if (ossKey.startsWith("oss://")) {
            String[] parts = ossKey.replace("oss://", "").split("/", 2);
            objectName = parts[1];
        } else if (ossKey.startsWith("https://")) {
            String[] parts = ossKey.replace("https://", "").split("/", 2);
            objectName = parts[1];
        } else {
            log.error("对象路径格式不正确，应以oss://或https://开头");
            throw new DataNotFoundException("未找到对应文件");
        }
        return ossClient.doesObjectExist(aliOssProperties.getBucketName(), objectName);
    }

    /**
     * 批量判断oss文件是否存在
     * @param ossKeys 文件ossKey集合
     * @return 文件集合是否存在
     */
    public boolean checkFilesExist(List<String> ossKeys) {
        for (String ossKey : ossKeys) {
            if (!checkFileExist(ossKey)) {
                return false;
            }
        }
        return true;
    }

    /**
     * 通用文件上传方法 - 上传到阿里云 OSS
     * @param file 上传的文件
     * @param targetDirectory OSS 目标目录
     * @param isPublic 是否公共读
     * @return 文件访问URL
     */
    private String uploadFile(MultipartFile file, String targetDirectory, boolean isPublic) {
        if (file.isEmpty()) {
            throw new ParamErrorException("文件不能为空");
        }

        // 获取文件扩展名
        String originalFilename = file.getOriginalFilename();
        String fileExtension = getFileExtension(originalFilename);
        String uniqueFileName = UUID.randomUUID().toString() + (fileExtension.isEmpty() ? "" : "." + fileExtension);

        // 构建 OSS 对象名称（包含目录路径）
        String objectName = targetDirectory + "/" + uniqueFileName;

        // 创建OSSClient实例
        OSS ossClient = new OSSClientBuilder().build(aliOssProperties.getEndpoint(), aliOssProperties.getAccessKeyId(), aliOssProperties.getAccessKeySecret());

        try {
            ObjectMetadata metadata = new ObjectMetadata();
            if (isPublic) {
                metadata.setObjectAcl(CannedAccessControlList.PublicRead);  // 公共读
            } else {
                metadata.setObjectAcl(CannedAccessControlList.Private);     // 私有
            }
            String contentType = file.getContentType();
            if (contentType != null) {
                metadata.setContentType(contentType);  // 设置文件类型
            }
            metadata.setContentLength(file.getSize());  // 文件字节数
            PutObjectRequest putObjectRequest = new PutObjectRequest(
                    aliOssProperties.getBucketName(),  // 存储桶名
                    objectName,                        // 文件名称
                    file.getInputStream(),             // 文件数据流
                    metadata                           // 文件元数据
            );
            // 上传文件到 OSS
            ossClient.putObject(putObjectRequest);

            // 生成文件访问URL
            String fileUrl = generateFileUrl(objectName, isPublic);

            return fileUrl;

        } catch (OSSException oe) {
            log.error("OSS服务异常: {}", oe.getErrorMessage(), oe);
            throw new FileUploadErrorException();
        } catch (ClientException ce) {
            log.error("OSS客户端异常: {}", ce.getMessage(), ce);
            throw new FileUploadErrorException();
        } catch (IOException e) {
            log.error("文件读取失败: {}", e.getMessage(), e);
            throw new FileUploadErrorException();
        } finally {
            if (ossClient != null) {
                ossClient.shutdown();
            }
        }
    }

    /**
     * 生成文件访问URL
     */
    private String generateFileUrl(String objectName, boolean isPublic) {
        if (isPublic) {
            // 公共读文件URL
            return String.format("https://%s.%s/%s",
                    aliOssProperties.getBucketName(),
                    aliOssProperties.getEndpoint(),
                    objectName);
        } else {
            // 私有文件返回标准OSS路径，便于后续生成签名URL
            return String.format("oss://%s/%s",
                    aliOssProperties.getBucketName(),
                    objectName);
        }
    }
}
```

### 3. 业务使用
现在我们已经成功整合了oss服务，可以在业务中使用了。

整合完成后，我们在业务中通常采用 **“前后端分离上传”** 的模式：

- **前端直传/上传**：用户在前端选择文件，调用专门的上传接口（或直传 OSS），获取文件的 URL 或 OSS Key。
- **业务提交**：前端调用业务接口（如“保存用户信息”）时，将获取到的 OSS Key 作为参数传递。
- **后端校验**：后端保存 OSS Key 到数据库，并可根据需要校验文件是否存在。

**优点**：
- **节省带宽**：文件流不经过你的应用服务器，极大减轻服务器压力。
- **响应快**：用户上传图片的同时可以填写表单，体验更好。

## 三、文件的清理
到此，我们已经正常打通了oss文件从上传到读取的流程，但是为了防止大量上传后无用的文件堆积影响内存，我们还需要处理文件的清理逻辑。

对于文件清除，我们可以根据业务来进行删除，例如用户更换头像则在更换后删除原头像文件，避免文件堆积。

不过这样只能处理正常业务流程清除，如果分离上传和业务api的话，用户上传后没有调用对应业务api，就产生了孤儿数据，占用内存。

### 解决方案：临时目录 + 定时清理

我们可以采用 **“两段式存储”** 策略：

1. **临时目录**：所有用户上传的文件，先统一放入 `temp` 目录下。
2. **转正逻辑**：当用户成功提交业务表单后，后端将文件从 `temp` 移动到正式目录。
3. **定时清理**：设置一个定时任务，扫描 `temp` 目录，删除创建时间超过 24 小时的文件。

这样，那些“上传了但没用”的孤儿文件就会被自动清理，既节省了空间，又保证了数据的一致性。

---
**总结**

OSS 是现代云原生应用不可或缺的基础设施。通过 Spring Boot 整合 OSS，不仅能解决文件存储难题，还能通过合理的架构设计（如私有读、临时目录清理）来保障数据的安全与整洁。