+++
title = "微信开发：如何实现AMR->MP3->AMR音频转码"
date = "2020-03-25 17:06:00"
url = "archives/688"
tags = ["Java","微信"]
categories = ["后端"]
+++

## 概述 ##

接触微信公众号开发的小伙伴一定对AMR这种音频格式不陌生，在微信公众号发送的语音，基本是这种格式，接收到后台后，还需要手动转换为MP3，这里探讨一下几种实现方式。

## 使用七牛提供的API ##

如果你使用七牛作为系统的云储存，那么音频转码将非常简单，先来看几个文档：  
1）普通音视频转码（avthumb）

> [https://developer.qiniu.com/dora/api/1248/audio-and-video-transcoding-avthumb][https_developer.qiniu.com_dora_api_1248_audio-and-video-transcoding-avthumb]

1.  持久化处理结果消息回调数据格式

> [https://developer.qiniu.com/dora/api/1294/persistent-processing-status-query-prefop][https_developer.qiniu.com_dora_api_1294_persistent-processing-status-query-prefop]

```java
public class QiniuTest {

    private final static String accessKey = "xxxx";
    private final static String secretKey = "xxx-xxxxxx";
    private final static String bucketName = "myfilestore1";

    public static void main(String[] args) throws QiniuException {
        Auth auth = Auth.create(accessKey, secretKey);
        String uploadToken = auth.uploadToken(bucketName);
        Configuration configuration = new Configuration();
        UploadManager uploadManager = new UploadManager(configuration);

        // 上传amr文件
        final String fileKey = "123";
        Response response = uploadManager.put(new File("/Users/wuwenze/Downloads/1.amr"), fileKey, uploadToken);
        System.out.println(response);


        // 将mp3文件转码为amr
        OperationManager operationManager = new OperationManager(auth, configuration);
        StringMap params = new StringMap()//
                .putNotEmpty("notifyURL", "https://baidu.com")// 回调通知
                .putWhen("force", 1, true)//
                .putNotEmpty("pipeline", "test_a");// 使用私有多媒体队列(https://portal.qiniu.com/dora/mps/test_a/index)
        String fops = "avthumb/mp3/ab/64k|saveas/" + UrlSafeBase64.encodeToString(bucketName + ":" + fileKey); 

        // 持久化结果ID
        String persistentId = operationManager.pfop(bucketName, fileKey, fops, params);
        System.out.println(persistentId);



        // 上传mp3文件
        final String fileKey1 = "456";
        Response response1 = uploadManager.put(new File("/Users/wuwenze/Downloads/2.mp3"), fileKey1, uploadToken);
        System.out.println(response1);


        // 将mp3文件转码为amr
        StringMap params1 = new StringMap()//
                .putNotEmpty("notifyURL", "https://baidu.com")// 回调通知
                .putWhen("force", 1, true)//
                .putNotEmpty("pipeline", "test_a");// 使用私有多媒体队列(https://portal.qiniu.com/dora/mps/test_a/index)
        String fops1 = "avthumb/amr/ab/64k|saveas/" + UrlSafeBase64.encodeToString(bucketName + ":" + fileKey1);
        // 持久化结果ID
        String persistentId1 = operationManager.pfop(bucketName, fileKey1, fops1, params1);
        System.out.println(persistentId1); 
    }
}
```

上面的代码演示了使用七牛上传文件->异步转码的过程，转码指令发出后，如果配置了notifyURL，那么七牛转码成功后，会主动推送结果过来，也可以用持久化结果ID，主动去查询转码的结果。

## 使用FFMPEG ##

如果没有使用云储存，也可以在自己的服务器上安装ffmpeg，来实现转码。  
官网地址：[http://ffmpeg.org/][http_ffmpeg.org] 自行按照操作系统安装之。

### MP3转换AMR
```bash
ffmpeg -i 1.mp3 -ac 1 -ar 8000 1.amr
```

### AMR转换MP3 
```bash
ffmpeg -i 1.amr 1.mp3
```

基于以上固定格式的命令，咱们可以封装一个简单的工具类：

```java
public class FFMpegUtils {
    private static final Logger log = Slf4j.getLogger();

    public static InputStream amrToMp3(InputStream source) {
        return doConvert(source, ".amr", ".mp3");
    }

    public static InputStream mp3ToAmr(InputStream source) {
        return doConvert(source, ".mp3", ".amr");
    }

    private static InputStream doConvert(InputStream source, String sourceSuffix, String targetSuffix) {
        final String tempDir = System.getProperty("java.io.tmpdir") + "/ffmpeg_temp/";
        final String tempFileName = UuidUtil.randomUuid();
        final File sourceFile = new File(tempDir, tempFileName + sourceSuffix);
        final File targetFile = new File(tempDir, tempFileName + targetSuffix);

        String command = null;
        try {
            if (!sourceFile.exists()) {
                File parentFolder = sourceFile.getParentFile();
                if (!parentFolder.exists()) {
                    parentFolder.mkdirs();
                }
                sourceFile.createNewFile();
            }

            // 将 source 转换成文件
            IoUtil.write(source, sourceFile);

            // MP3转换AMR： ffmpeg -i 1.mp3 -ac 1 -ar 8000 1.amr
            if (StringUtil.equals(sourceSuffix, ".mp3") && StringUtil.equals(targetSuffix, ".amr")) {
                command = String.format("ffmpeg -i %s -ac 1 -ar 8000 %s", //
                    FileUtil.getCanonicalPath(sourceFile), FileUtil.getCanonicalPath(targetFile));
            }
            // AMR转换MP3： ffmpeg -i 1.amr 1.mp3
            else if (StringUtil.equals(sourceSuffix, ".amr") && StringUtil.equals(targetSuffix, ".mp3")) {
                command = String.format("ffmpeg -i %s %s", //
                    FileUtil.getCanonicalPath(sourceFile), FileUtil.getCanonicalPath(targetFile));
            } else {
                throw new RuntimeException("#0325 FFMpegUtils.doConvert() error, 不支持的文件格式，sourceSuffix=" + sourceSuffix + ",targetSuffix=" + targetSuffix);
            }

            Runtime runtime = Runtime.getRuntime();
            Process process = runtime.exec(command);
            runtime.addShutdownHook(new ProcessKiller(process));
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(process.getErrorStream()));
            String line;
            while ((line = BufferedReaderUtil.readLine(bufferedReader)) != null) {
                log.info("#0325 FFMpegUtils.doConvert(): " + line);
            }
        } catch (Throwable e) {
            throw new RuntimeException("#0325 FFMpegUtils.doConvert() error, command=" + command, e);
        }
        // 返回转换的 targetFile
        ByteArrayOutputStream targetStream = new ByteArrayOutputStream();
        IoUtil.write(IoUtil.fileInputStream(targetFile), targetStream);
        return new ByteArrayInputStream(targetStream.toByteArray());
    }

    public static void main(String[] args) throws IOException {
        // test mp3 -> amr
        File mp3File = new File("/Users/wuwenze/Downloads/1.mp3");
        InputStream amrStream = FFMpegUtils.mp3ToAmr(IoUtil.fileInputStream(mp3File));

        File amrFile = new File("/Users/wuwenze/Downloads/1-mp3-amr-result.amr");
        if (amrFile.createNewFile()) {
            IoUtil.write(amrStream, amrFile);
        }

        // test amr -> mp3
        InputStream mp3Stream = FFMpegUtils.amrToMp3(IoUtil.fileInputStream(amrFile));
        File newMp3File = new File("/Users/wuwenze/Downloads/1-amr-mp3-result.mp3");
        if (newMp3File.createNewFile()) {
            IoUtil.write(mp3Stream, newMp3File);
        }
    }
}
```

在自己的服务器上使用ffmpeg转码很简单，但是性能损耗也很大，如果能使用云存储，可能是更好的选择。


[https_developer.qiniu.com_dora_api_1248_audio-and-video-transcoding-avthumb]: https://developer.qiniu.com/dora/api/1248/audio-and-video-transcoding-avthumb
[https_developer.qiniu.com_dora_api_1294_persistent-processing-status-query-prefop]: https://developer.qiniu.com/dora/api/1294/persistent-processing-status-query-prefop
[http_ffmpeg.org]: http://ffmpeg.org/