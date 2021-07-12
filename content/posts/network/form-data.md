---
title: Golang 使用 multipart/form-data 实现上传与下载
author: ash
tags: ["Go", "form-data", "HTTP"]
categories: ["记录一下"]
date: 2021-06-13T13:27:09+08:00
image: chino03.jpg
---

# 0x01 Form（表单）

`From` 是HTML中的重要语法元素.

一个 `Form` 不仅包含文本内容、标记等，还包含被称为控件的特殊元素. 

用户通过修改控件填写表单，然后将表单数据以HTTP的 Get/Post 请求的形式 提交(submit) 给Web服务器.


# 0x02 Form的标准格式

在HTML的文档中，一个标准的Form格式如下:

```html
<form action="http://ip:port/xxx" method="get">
   <input type="text" name="language" value="go" />
   <input type="text" name="author" value="ash" />
   <input type="submit" />
</form>
```

`Form`的 method 为 `get` 时，HTTP请求会将`Form`的输入文本作为查询字符串参数.（?language=go&author=ash）

```html
▼ General
    Request URL: http://ip:port/xxx?language=go&author=ash
    Request Method: GET
...
```


`Form`的method也可以为 `post`：

```html
<form action="http://ip:8080/xxx" method="post">
   <input type="text" name="language" value="go" />
   <input type="text" name="author" value="ash" />
   <input type="submit" />
</form>
```

改为`post`的`Form`表单在提交后发出的HTTP请求与method为get时的请求不同，此时`Form`的参数不会再以查询字符串参数的形式放在请求的URL中，而是会被写入`HTTP`的`BODY`中.

```html
▼ General
    Request URL: http://ip:port/xxx
    Request Method: POST
    Content-Type: application/x-www-form-urlencoded
    ...
▼ Form Data
    language=go&author=ash
```

`HTTP`请求Headers中我们发现新增了一个字段：`Content-Type`，在这里例子中，它的值为`application/x-www-form-urlencoded`. 

在`Form`中可以使用`enctype`属性改变`Form`传输数据的内容编码类型.

`enctype`的其他可选值:

* text/plain
* multipart/form-data
* ...

采用get方法的`Form`，表单参数以查询字符串参数的形式放入HTTP请求中，这使得其应用场景相对局限.

1. 当参数值很多，参数值很长时，可能会超出URL最大长度限制
2. 传递敏感数据时，参数值以明文放在HTTP请求头不安全
3. 无法胜任传递二进制数据(比如一个文件内容)的情形
4. ...


因此，在面对上述这些情形时，post方法提交表单更有优势.
当`enctype`为不同值时，post方法提交的表单在`HTTP Body`中传输的数据形式如下：

当`enctype`为`application/x-www-form-urlencoded`时,，content格式就如上文所示的一样，数据呈现为`key1=value1&key2=value2&…`的形式

```html
▼ Form Data
    language=go&author=ash
```

当`enctype`为`text/plain`时，这种编码格式也称为raw，即将数据内容原封不动的放入Body中传输，保持数据的原先的编码方式(通常为utf-8).

```html
▼ Request Payload
    language=go
    author=ash
```

而当`enctype`为`multipart/form-data`时，`HTTP Body`中的数据以多段(part)的形式呈现，段与段之间使用指定的随机字符串分隔，该随机字符串也会随着`HTTP Post`请求一并传给服务端.

```html
▼ General
    Request URL: http://ip:port/xxx
    Request Method: POST
    Content-Type: multipart/form-data; boundary=--FormBoundaryEVASVNX99ijc6X1ZxE
▼ Form Data
    --FormBoundaryEVASVNX99ijc6X1ZxE
    Content-Disposition: form-data; name="language"

    go
    --FormBoundaryEVASVNX99ijc6X1ZxE
    Content-Disposition: form-data; name="author"

    ash
    --FormBoundaryEVASVNX99ijc6X1ZxE--
```

服务端接收到这些数据时，根据分段`Content-Type`的指示，便可以有针对性的对分段数据进行解析了.

文件分段默认为`Content-Type为text/plain`，对于无法识别的文件类型，`Content-Type`通常会设置为`application/octet-stream`

# 支持以multipart/form-data格式上传文件的Go服务器

golang的`http.Request`模块提供了`ParseMultipartForm`方法对以`multipart/form-data`格式传输的数据进行解析，解析即是将数据映射为Request结构的MultipartForm字段的过程.

```go
// $GOROOT/src/net/http/request.go

type Request struct {
    ... ...
    // MultipartForm is the parsed multipart form, including file uploads.
    // This field is only available after ParseMultipartForm is called.
    // The HTTP client ignores MultipartForm and uses Body instead.
    MultipartForm *multipart.Form
    ... ...
}
```

multipart.Form代表了一个解析后的multipart/form-data的Body，其结构如下:

```go
// $GOROOT/src/mime/multipart/formdata.go

// Form is a parsed multipart form.
// Its File parts are stored either in memory or on disk,
// and are accessible via the *FileHeader's Open method.
// Its Value parts are stored as strings.
// Both are keyed by field name.
type Form struct {
    Value map[string][]string
    File  map[string][]*FileHeader
}
```

可以看到`Form`结构由两个map组成，一个map中存放了所有的value part，另外一个map存放了所有的file part.

value part集合比较简单，map中的key就是每个值分段中的"name". 

每个file part对应一组FileHeader，FileHeader的结构如下：

```go
// $GOROOT/src/mime/multipart/formdata.go
type FileHeader struct {
    Filename string
    Header   textproto.MIMEHeader
    Size     int64

    content []byte
    tmpfile string
}
```

每个file part的FileHeader包含五个字段:

1. Filename – 上传文件的原始文件名
2. Size – 上传文件的大小（单位：字节）
3. content – 内存中存储的上传文件的（部分或全部）数据内容
4. tmpfile – 在服务器本地的临时文件中存储的部分上传文件的数据内容(如果上传的文件大小大于传给ParseMultipartForm的参数maxMemory，剩余部分存储在临时文件中)
5. Header – file part的header内容，它也是一个map，结构如下:

```go
// $GOROOT/src/net/textproto/header.go

// A MIMEHeader represents a MIME-style header mapping
// keys to sets of values.
type MIMEHeader map[string][]string
```

有了上述对通过`multipart/form-data`格式上传文件的原理的拆解，我们就可以使用Go实现一个简单的支持以`multipart/form-data`格式上传文件的Go服务器.

```go

```