# KCos SDK Guidance

鉴于我们维护能力实在有限，无法为所有的编程语言都提供相对应的SDK，因此开源出来SDK的接口，各语言可以利用HttpClient构造请求来完成对应的功能。

访问接口，必须要在接口上附带X-AppId和X-AppKey字段，所有接口都要带上！后面所有的接口里不会再重复说明这个字段了。

## 用户管理

 Cos SDK的用户与调用方的客户ID是不一致的，需要用一个接口将这两个数据联系起来。只要客服提供一个userTag，服务端返回一个uint类型的userId。

一个App底下的同一个userTag对应的userId是一致的，这个接口会返回一样的值。

```http
https://cos.kevinc.ltd:8082/user/createAppUser
```

### Request

```json
{
    "userTag":"xxxxxx"
}
```

### Response

```json
{
    "userTag": "xxxxxxxxx",
    "userId": 111111
}
```

## 文件管理

文件上传的流程，必须要先创建文件的句柄，拿到文件的分块信息后，再进行上传。

## 创建文件句柄

提交一些必要的元信息，用于创建文件的句柄。需要在头部附带UserId的信息。

```http
https://cos.kevinc.ltd:8082/file/createFileEntry

X-UserId 1111111
```

### Request

```json
{
    "path": "/yourParentPath/relatedPath",
    "fileNameWithExt": "file.ext",
    "fileSize": 11111111, // uint64类型
    "sha256": "xxxxxxxxxxxxxxxxxxx",
    "mimeType": "application/octet-stream", // 可不提供，默认值就是这个
    "deadLine": "2022-12-31T00:00:00.000+08:00", // 文件过期时间，可不提供，不提供默认永久有效
    "protection": 0, // 见下方文档说明，默认为0
    "securityPayload": ""
}
```

protection字段有4个取值：

0，表示文件公共读

1，域内可见（同一App下可见）

2，文件仅上传者自己可看

3，提供访问密码后可见。然后把密码字段写在securityPayload中。

4，向指定地址构造请求后可见，将指定地址写在securityPayload中，这个地址必须为有效的https链接，并且在地址中不可以附带query parameter。服务器会向这个地址发送一个post请求，结构如下。返回状态码为2xx时，会允许下载，其他情况则不允许下载。

### Response

```json
{
    "id": 37, // 文件Id，非常重要，需要保存下来，uint64类型
    "fileSize": 11111111,
    "frames": 11, // 文件分块的数量
    "nextRequestedFrame": 1, // 见文档
    "pathHierarchy": [
        "yourParentPath",
        "relatedPath"
    ],
    "fileNameWithExt": "file.ext",
    "creationTime": "2022-09-29T15:17:49.0336106+00:00",
    "deadLine": "2022-12-31T00:00:00+08:00"
}
```

nextRequestedFrame表示下一次需要进行传送的帧标号。是一个uint32类型，若这个值是0，表示这个文件的二进制已经在系统中出现过了，会重复利用，而无需浪费流量进行传送。

## 上传文件块

将文件分为1MB的包，进行发送，每个文件块最大1MB。

```http
https://cos.kevinc.ltd:8082/file/upload?fileId=xxx&seqNumber=1

X-UserId 1111111
```

seqNumber表示文件块的顺序，以1为基准，每个包大小为1MB（除了最后一个包可以不是）请求体就直接放二进制就行了

### Response

```json
{
    "nextRequestedFrame": 2
}
```

nextRequestedFrame表示下一次需要进行传送的帧标号。是一个uint32类型，若这个值是0，表示传输过程已经结束了
