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

# 0x03 文件上传服务

## 支持以multipart/form-data格式上传文件的Go服务器

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
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
)

const uploadPath = "./upload"

func handleUploadFile(w http.ResponseWriter, r *http.Request) {
	r.ParseMultipartForm(100)
	mForm := r.MultipartForm

	for k := range mForm.File {
		// k is the key of file part
		file, fileHeader, err := r.FormFile(k)
		if err != nil {
			fmt.Println("inovke FormFile error:", err)
			return
		}
		defer file.Close()
		fmt.Printf("the uploaded file: name[%s], size[%d], header[%#v]\n",
			fileHeader.Filename, fileHeader.Size, fileHeader.Header)

		// store uploaded file into local path
		localFileName := uploadPath + "/" + fileHeader.Filename
		out, err := os.OpenFile(localFileName, os.O_RDWR|os.O_CREATE|os.O_APPEND, 0666)
		// out, err := os.Create(localFileName)
		if err != nil {
			fmt.Printf("failed to open the file %s for writing", localFileName)
			return
		}
		defer out.Close()
		_, err = io.Copy(out, file)
		if err != nil {
			fmt.Printf("copy file err:%s\n", err)
			return
		}
		fmt.Printf("file %s uploaded ok\n", fileHeader.Filename)
	}
}

func main() {
	http.HandleFunc("/upload", handleUploadFile)
	http.ListenAndServe(":8081", nil)
}
```

运行Go文件上传服务:

```sh
mkdir upload
# 启动文件服务
go run fileserver.go
```

使用curl/httpie/curlie/ht/postman等http工具：

```sh
http -f --follow POST :8081/upload file1@/path-to/test1

或者

curl --location --request POST 'http://localhost:8081/upload' --form 'file1=@"/path-to/test2"'

或者

...
```

可以看到服务终端打印:

```sh
go run fileserver.go

the uploaded file: name[test1], size[21634], header[textproto.MIMEHeader{"Content-Disposition":[]string{"form-data; name=\"file1\"; filename=\"test1\""}, "Content-Type":[]string{"text/markdown"}}]
file test1 uploaded ok
```

然后可以看到 文件上传服务成功将接收到的文件存入upload目录下：

```sh
tree upload/

upload/
├── test1
├── test2
└── test3

0 directories, 3 files
```

##  支持以multipart/form-data格式上传文件的Go客户端

别人的东西再好，终究不如自己的顺手~ 所以我们自己构建一个客户端.

使用mime/multipart包，很容易可以构建符合multipart/form-data格式的HTTP请求.

```go
package main

import (
	"bytes"
	"flag"
	"fmt"
	"io"
	"mime/multipart"
	"net/http"
	"os"
	"path/filepath"
)

var (
	filePath string
	addr     string
)

func init() {
	flag.StringVar(&filePath, "file", "", "the file to upload")
	flag.StringVar(&addr, "addr", "localhost:8081", "the addr of file server")
	flag.Parse()
}

func main() {
	if filePath == "" {
		fmt.Println("file must not be empty")
		return
	}

	err := doUpload(addr, filePath)
	if err != nil {
		fmt.Printf("upload file [%s] error: %s", filePath, err)
		return
	}
	fmt.Printf("upload file [%s] ok\n", filePath)
}

func createReqBody(filePath string) (string, io.Reader, error) {
	var err error

	buf := new(bytes.Buffer)
	bw := multipart.NewWriter(buf) // body writer

	f, err := os.Open(filePath)
	if err != nil {
		return "", nil, err
	}
	defer f.Close()

	// text part1
	p1w, _ := bw.CreateFormField("name")
	p1w.Write([]byte("Ash XYZ"))

	// text part2
	p2w, _ := bw.CreateFormField("age")
	p2w.Write([]byte("15"))

	// file part1
	_, fileName := filepath.Split(filePath)
	fw1, _ := bw.CreateFormFile("file1", fileName)
	io.Copy(fw1, f)

	bw.Close() //write the tail boundry
	return bw.FormDataContentType(), buf, nil
}

func doUpload(addr, filePath string) error {
	// create body
	contType, reader, err := createReqBody(filePath)
	if err != nil {
		return err
	}

	url := fmt.Sprintf("http://%s/upload", addr)
	req, err := http.NewRequest("POST", url, reader)

	// add headers
	req.Header.Add("Content-Type", contType)

	client := &http.Client{}
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println("request send error:", err)
		return err
	}
	resp.Body.Close()
	return nil
}

```

使用go客户端上传文件

```sh
go run client.go -file /path-to/test4
```


# 0x03 自定义file分段中的header

...

# 0x04 大文件上传问题

之前的客户端程序中，我们构建HTTP Body时，使用了`bytes.Buffer`加载待上传文件的所有内容. 但是当文件很大的时候，内存空间消耗巨大甚至不足以容纳文件大小~

使用golang提供的io.Pipe，可以较好的解决这个问题~

```go
var quoteEscaper = strings.NewReplacer("\\", "\\\\", `"`, "\\\"")

func escapeQuotes(s string) string {
	return quoteEscaper.Replace(s)
}

func createReqBody(filePath string) (string, io.Reader, error) {
    var err error
    pr, pw := io.Pipe()
    bw := multipart.NewWriter(pw) // body writer
    f, err := os.Open(filePath)
    if err != nil {
        return "", nil, err
    }

    go func() {
        defer f.Close()
        // text part1
        p1w, _ := bw.CreateFormField("name")
        p1w.Write([]byte("Tony Bai"))

        // text part2
        p2w, _ := bw.CreateFormField("age")
        p2w.Write([]byte("15"))

        // file part1
        _, fileName := filepath.Split(filePath)
        h := make(textproto.MIMEHeader)
        h.Set("Content-Disposition",
            fmt.Sprintf(`form-data; name="%s"; filename="%s"`,
                escapeQuotes("file1"), escapeQuotes(fileName)))
        h.Set("Content-Type", "application/tar")
        fw1, _ := bw.CreatePart(h)
        cnt, _ := io.Copy(fw1, f)
        log.Printf("copy %d bytes from file %s in total\n", cnt, fileName)
        bw.Close() //write the tail boundry
        pw.Close()
    }()
    return bw.FormDataContentType(), pr, nil
}
```

在这个方案中，我们通过io.Pipe函数创建了一个读写管道，写入端作为io.Writer实例传给multipart.NewWriter，读取端返回给调用者，用于构建HTTP请求时使用. 

io.Pipe基于channel实现，其内部不维护任何内存缓存:

```go
// Pipe creates a synchronous in-memory pipe.
// It can be used to connect code expecting an io.Reader
// with code expecting an io.Writer.
//
// Reads and Writes on the pipe are matched one to one
// except when multiple Reads are needed to consume a single Write.
// That is, each Write to the PipeWriter blocks until it has satisfied
// one or more Reads from the PipeReader that fully consume
// the written data.
// The data is copied directly from the Write to the corresponding
// Read (or Reads); there is no internal buffering.
//
// It is safe to call Read and Write in parallel with each other or with Close.
// Parallel calls to Read and parallel calls to Write are also safe:
// the individual calls will be gated sequentially.
func Pipe() (*PipeReader, *PipeWriter) {
	p := &pipe{
		wrCh: make(chan []byte),
		rdCh: make(chan int),
		done: make(chan struct{}),
	}
	return &PipeReader{p}, &PipeWriter{p}
}
```

通过Pipe返回的reader端在读取管道中的数据时，如果尚未有数据写入管道，那么读端会阻塞.

由于HTTP请求在被发送时(client.Do(req))才会真正基于构建req时传入的reader对Body数据进行读取，因此client会阻塞在对管道的read上. 因此我们不能将读写两端的操作放在同一个goroutine中，那样所有goroutine都会因挂起而panic.

在上面代码中，函数createReqBody内部创建了一个新goroutine，将真正构建multipart/form-data body的工作放在了新goroutine中，新goroutine最终会将待上传文件的数据通过管道writer端写入管道.

```go
count, _ := io.Copy(partWriter, fileReader)
```

而这些数据也会被client读取并通过网络连接传输出去.  io.Copy的实现如下:

```go
// Copy copies from src to dst until either EOF is reached
// on src or an error occurs. It returns the number of bytes
// copied and the first error encountered while copying, if any.
//
// A successful Copy returns err == nil, not err == EOF.
// Because Copy is defined to read from src until EOF, it does
// not treat an EOF from Read as an error to be reported.
//
// If src implements the WriterTo interface,
// the copy is implemented by calling src.WriteTo(dst).
// Otherwise, if dst implements the ReaderFrom interface,
// the copy is implemented by calling dst.ReadFrom(src).
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}
```

io.copyBuffer内部维护了一个默认32k的小buffer，它每次从src尝试最大读取32k的数据，并写入到dst中，直到读完为止. 这样无论待上传的文件有多大，我们实际上每次上传所分配的内存仅有32k.

如果觉得32k仍然很大，每次上传要使用更小的buffer，可以用io.CopyBuffer替代io.Copy:

```go
func createReqBody(filePath string) (string, io.Reader, error) {
	var err error
	pr, pw := io.Pipe()
	bw := multipart.NewWriter(pw) // body writer
	f, err := os.Open(filePath)
	if err != nil {
		return "", nil, err
	}

	go func() {
		defer f.Close()
		// text part1
		p1w, _ := bw.CreateFormField("name")
		p1w.Write([]byte("Tony Bai"))

		// text part2
		p2w, _ := bw.CreateFormField("age")
		p2w.Write([]byte("15"))

		// file part1
		_, fileName := filepath.Split(filePath)
		h := make(textproto.MIMEHeader)
		h.Set("Content-Disposition",
			fmt.Sprintf(`form-data; name="%s"; filename="%s"`,
				escapeQuotes("file1"), escapeQuotes(fileName)))
		h.Set("Content-Type", "application/tar")
		fw1, _ := bw.CreatePart(h)
		var buf = make([]byte, 1024)
		cnt, _ := io.CopyBuffer(fw1, f, buf)
		log.Printf("copy %d bytes from file %s in total\n", cnt, fileName)
		bw.Close() //write the tail boundry
		pw.Close()
	}()
	return bw.FormDataContentType(), pr, nil
}
```

# 0x05 参考文献

1. [https://golangnote.com/topic/246.html](https://golangnote.com/topic/246.html)
2. [https://www.ietf.org/rfc/rfc7233.txt](https://www.ietf.org/rfc/rfc7233.txt)