---
title: "SpringBoot实现大文件上传"
date: "2023-07-03"
tags: ['SpringBoot']
categories: ['动手实践']
---

大文件上传是提高用户体验感的重要功能，像百度网盘、阿里网盘等等都支持断点续传和文件秒传功能，减少了网络波动和网络带宽对文件的限制。

- 「文件分块」：将大文件拆分成小文件，将小文件上传\下载，最后再将小文件组装成大文件；
- 「断点续传」：在文件分块的基础上，将每个小文件采用单独的线程进行上传\下载，如果碰到网络故障，可以从已经上传\下载的部分开始继续上传\下载未完成的部分，而没有必要从头开始上传\下载；
- 「文件秒传」：资源服务器中已经存在该文件，其他人上传时直接返回该文件的URI。

本文将介绍SpringBoot中实现大文件上传的主要代码：

## 文件分块

文件分块需要在前端处理，前端通过获取文件的唯一`md5`的值来区分不同文件，通过确定文件分块的大小和分块的数量，为每一个分块指定一个索引值后上传不同块文件。

通过`md5`也可以用来校验服务器上是否存在该文件以及文件的上传状态。

- 如果文件存在，直接返回文件地址；
- 如果文件不存在，但是有上传状态，即部分分块上传成功，则返回未上传的分块索引数组；
- 如果文件不存在，且上传状态为空，则所有分块均需要上传。

```javascript
const readFileMD5 = () => {
    let fileReader = new FileReader();
    fileReader.readAsBinaryString(file)
    fileReader.addEventListener("load", (e) => {
        let fileBlob = e.target.result
        fileMD5 = md5(fileBlob)
        const formData = new FormData();
        formData.append("md5", fileMD5)
        axios
            .post("http://localhost:9191/fileUpload/checkFileMd5", formData)
            .then((res) => {
                if (res.data.message == "文件已存在") {
                    //文件已存在不走后面分片了，直接返回文件地址到前台页面
                    console.log(res.data)
                } else {
                    //文件不存在存在两种情况，一种是返回data：null代表未上传过 一种是data:[xx，xx] 还有哪几片未上传
                    if (res.data.data != null) {
                        //还有几片未上传情况，断点续传
                        chunkArr = res.data.data;
                    }
                    readChunkMD5();
                }
            })
            .catch((e) => {
            })
    })
}
```

通过`slice`方法来取出索引在文件中对应位置的分块。

```javascript
const getChunkInfo = (file, currentChunk, chunkSize) => {
    let start = currentChunk * chunkSize
    let end = Math.min(file.size, start + chunkSize)
    let chunk = file.slice(start, end)
    return {start, end, chunk}
}
```

## 断点续传、文件秒传

通过`redis`来存储上传文件的状态和上传文件的地址。

如果文件完整上传，返回文件路径；如果部分上传则返回未上传的分块数组；如果未上传过返回提示信息。

> Note:
> 在上传分块时会产生两个文件，一个是文件主体，一个是临时文件。临时文件可以看做是一个数组文件，为每一个分块分配一个值为127的字节。

校验MD5值时会用到两个值：

- 文件上传状态：只要该文件上传过就不为空，如果完整上传则为true，部分上传返回false；
- 文件上传地址：如果文件完整上传，返回文件路径；部分上传返回临时文件路径。

```java
@PostMapping("/checkFileMd5")
@ResponseBody
public Result checkFileMd5(String md5) throws IOException {
    Object processingObj = stringRedisTemplate.opsForHash().get(UploadConstants.FILE_UPLOAD_STATUS, md5);
    if (processingObj == null) {
        return Result.ok("该文件没有上传过");
    }
    boolean processing = Boolean.parseBoolean(processingObj.toString());
    String value = stringRedisTemplate.opsForValue().get(UploadConstants.FILE_MD5_KEY + md5);
    if (processing) {
        return Result.ok(value, "文件已存在");
    } else {
        File confFile = new File(value);
        byte[] completeList = FileUtil.readBytes(confFile);
        List<Integer> missChunkList = new LinkedList<>();
        for (int i = 0; i < completeList.length; i++) {
            if (completeList[i] != Byte.MAX_VALUE) {
                missChunkList.add(i);
            }
        }
        return Result.ok(missChunkList, "该文件上传了一部分");
    }
}
```

## 分块上传、文件合并

上面利用文件的`md5`值来维护分块和文件的关系，因此我们会将具有相同`md5`值的分块进行合并，由于每个分块都有自己的索引值，所以我们会将分块按索引像插入数组一样分别插入文件中，形成完整的文件。

分块上传时，要和前端的分块大小、分块数量、当前分块索引等对应好，以备文件合并时使用，此处我们采用的是**磁盘映射**的方式来合并文件。

```java
public boolean uploadFileByMappedByteBuffer(MultipartFileDTO multipartFileDTO) throws IOException {
    // 路径：服务器存储地址/md5字符串
    String uploadDirPath = finalDirPath + multipartFileDTO.getMd5();
    File tmpDir = new File(uploadDirPath);
    if (!tmpDir.exists()) {
        tmpDir.mkdirs();
    }

    String fileName = multipartFileDTO.getName();
    // 临时文件名
    String tempFileName = fileName + "_tmp";
    File tmpFile = new File(uploadDirPath, tempFileName);

    // 读操作和写操作都是允许的
    RandomAccessFile tempRaf = new RandomAccessFile(tmpFile, "rw");
    //它返回的就是nio通信中的file的唯一channel
    FileChannel fileChannel = tempRaf.getChannel();

    // 写入该分片数据   分片大小 * 第几块分片获取偏移量
    long offset = CHUNK_SIZE * multipartFileDTO.getChunk();
    // 分片文件大小
    byte[] fileData = multipartFileDTO.getFile().getBytes();
    // 将文件的区域直接映射到内存
    MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, offset, fileData.length);
    mappedByteBuffer.put(fileData);
    // 释放
    FileMD5Util.freedMappedByteBuffer(mappedByteBuffer);
    fileChannel.close();

    boolean isOk = checkAndSetUploadProgress(multipartFileDTO, uploadDirPath);
    if (isOk) {
        boolean flag = renameFile(tmpFile, fileName);
        System.out.println("upload complete !!" + flag + " name=" + fileName);
        return flag;
    }
    return false;
}

public boolean renameFile(File toBeRenamed, String toFileNewName) {
    // 检查要重命名的文件是否存在，是否是文件
    if (!toBeRenamed.exists() || toBeRenamed.isDirectory()) {
        log.info("File does not exist: " + toBeRenamed.getName());
        return false;
    }
    String p = toBeRenamed.getParent();
    File newFile = new File(p + File.separatorChar + toFileNewName);
    // 修改文件名
    return toBeRenamed.renameTo(newFile);
}
```

每当完成一次分块的上传，还需要去检查文件的上传进度，看文件是否上传完成。

```java
private boolean checkAndSetUploadProgress(MultipartFileDTO multipartFileDTO, String uploadDirPath) throws IOException {
    String fileName = multipartFileDTO.getName();
    // 路径/filename.conf
    File confFile = new File(uploadDirPath, fileName + ".conf");
    RandomAccessFile accessConfFile = new RandomAccessFile(confFile, "rw");
    // 把该分段标记为 true 表示完成
    System.out.println("set part " + multipartFileDTO.getChunk() + " complete");
    accessConfFile.setLength(multipartFileDTO.getChunks());
    accessConfFile.seek(multipartFileDTO.getChunk());
    accessConfFile.write(Byte.MAX_VALUE);

    // completeList 检查是否全部完成,如果数组里是否全部都是(全部分片都成功上传)
    byte[] completeList = FileUtil.readBytes(confFile);
    byte isComplete = Byte.MAX_VALUE;
    for (int i = 0; i < completeList.length && isComplete == Byte.MAX_VALUE; i++) {
        //与运算, 如果有部分没有完成则 isComplete 不是 Byte.MAX_VALUE
        isComplete = (byte) (isComplete & completeList[i]);
        System.out.println("check part " + i + " complete?:" + completeList[i]);
    }

    accessConfFile.close();
    // 更新redis中的状态：如果是true的话证明是已经该大文件全部上传完成
    if (isComplete == Byte.MAX_VALUE) {
        stringRedisTemplate.opsForHash().put(UploadConstants.FILE_UPLOAD_STATUS, multipartFileDTO.getMd5(), "true");
        stringRedisTemplate.opsForValue().set(UploadConstants.FILE_MD5_KEY + multipartFileDTO.getMd5(), uploadDirPath + "/" + fileName);
        return true;
    } else {
        if (!stringRedisTemplate.opsForHash().hasKey(UploadConstants.FILE_UPLOAD_STATUS, multipartFileDTO.getMd5())) {
            stringRedisTemplate.opsForHash().put(UploadConstants.FILE_UPLOAD_STATUS, multipartFileDTO.getMd5(), "false");
        }
        if (!stringRedisTemplate.hasKey(UploadConstants.FILE_MD5_KEY + multipartFileDTO.getMd5())) {
            stringRedisTemplate.opsForValue().set(UploadConstants.FILE_MD5_KEY + multipartFileDTO.getMd5(), uploadDirPath + "/" + fileName + ".conf");
        }
        return false;
    }
}
```

在MappedByteBuffer释放后再对它进行读操作的话就会引发jvm crash，在并发情况下很容易发生正在释放时另一个线程正开始读取，于是crash就发生了。所以为了系统稳定性释放前一般需要检查是否还有线程在读或写。

```java
public static void freedMappedByteBuffer(final MappedByteBuffer mappedByteBuffer) {
    try {
        if (mappedByteBuffer == null) {
            return;
        }

        mappedByteBuffer.force();
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Method getCleanerMethod = mappedByteBuffer.getClass().getMethod("cleaner", new Class[0]);
                    getCleanerMethod.setAccessible(true);
                    sun.misc.Cleaner cleaner = (sun.misc.Cleaner) getCleanerMethod.invoke(mappedByteBuffer,
                            new Object[0]);
                    cleaner.clean();
                } catch (Exception e) {
                    log.error("clean MappedByteBuffer error!!!", e);
                }
                log.info("clean MappedByteBuffer completed!!!");
                return null;
            }
        });

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

## 技术点

### RandomAccessFile

RandomAccessFile相当于是FileInputStream与FileOutputStream的封装结合，即可以读也可以写，并且RandomAccessFile支持移动到文件指定位置处开始读或写。

### MappedByteBuffer

MappedByteBuffer是Java提供的基于操作系统虚拟内存映射（MMAP）技术的文件读写API，底层不再通过read、write、seek等系统调用实现文件的读写。

### AccessController.doPrivileged()

`AccessController.doPrivileged`是一个在`AccessController`类中的静态方法，允许在一个类实例中的代码通知这个`AccessController`：它的代码主体是享受"privileged(特权的)"，它单独负责对它的可得的资源的访问请求，而不管这个请求是由什么代码所引发的。
这就是说，一个调用者在调用`doPrivileged`方法时，可被标识为 "特权"。在做访问控制决策时，如果`checkPermission`方法遇到一个通过`doPrivileged`调用而被表示为 "特权"的调用者，并且没有上下文自变量，`checkPermission`方法则将终止检查。如果那个调用者的域具有特定的许可，则不做进一步检查，`checkPermission`安静地返回，表示那个访问请求是被允许的；如果那个域没有特定的许可，则象通常一样，一个异常被抛出。


> 本文参考自：
> https://gitee.com/zhangxiaoQ/large-file-upload.git