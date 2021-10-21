# やっぱりzstdのほうがよかった　GoでZIP内部のzstdを解凍

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/Go-ZSTD/blob/main/ZIP_ZSTD.ipynb)

[以前の記事](https://www.soliton-cyber.com/blog/go-bzip2)で、zipの圧縮にbzip2を使ったものをGo言語で読み出す方法について記載しました。しかし、zstdのほうが高性能ということなので同じことをzstdでやってみます。


まずサンプルを用意します。


```shell
!man man > man.txt
!wc man.txt
```

      724  4977 38134 man.txt


zstdでzipアーカイブを作るには、zipfile-zstdモジュールを使うので、インストールします。



```shell
!pip install zipfile-zstd
```

    Collecting zipfile-zstd
      Downloading zipfile_zstd-0.0.3-py3-none-any.whl (4.1 kB)
    Collecting zstandard>=0.15.0
      Downloading zstandard-0.15.2-cp37-cp37m-manylinux2014_x86_64.whl (2.2 MB)
    [K     |████████████████████████████████| 2.2 MB 7.9 MB/s 
    [?25hInstalling collected packages: zstandard, zipfile-zstd
    Successfully installed zipfile-zstd-0.0.3 zstandard-0.15.2


これを使ってアーカイブを作るのは以下です。


```python
import zipfile_zstd as zipfile
with zipfile.ZipFile('man.zip', 'w', zipfile.ZIP_ZSTANDARD, compresslevel=19) as zfile:
    zfile.write('man.txt', 'man.txt')

!zipinfo man.zip
```

    Archive:  man.zip
    Zip file size: 12162 bytes, number of entries: 1
    -rw-r--r--  6.3 unx    38134 b- u093 21-Oct-06 08:54 man.txt
    1 file, 38134 bytes uncompressed, 12050 bytes compressed:  68.4%


bzip2では70.2%でしたので、少し良くなっています。さらにzstdの方が解凍速度がすごく速いらしい。

次にGo言語をインストールします。


```shell
!wget https://golang.org/dl/go1.17.1.linux-amd64.tar.gz
!tar -C /usr/local -xzf go1.17.1.linux-amd64.tar.gz

import os
os.environ['PATH'] += ":/usr/local/go/bin"
```

    --2021-10-06 08:55:05--  https://golang.org/dl/go1.17.1.linux-amd64.tar.gz
    Resolving golang.org (golang.org)... 74.125.142.141, 2607:f8b0:400e:c01::8d
    Connecting to golang.org (golang.org)|74.125.142.141|:443... connected.
    HTTP request sent, awaiting response... 302 Found
    Location: https://dl.google.com/go/go1.17.1.linux-amd64.tar.gz [following]
    --2021-10-06 08:55:05--  https://dl.google.com/go/go1.17.1.linux-amd64.tar.gz
    Resolving dl.google.com (dl.google.com)... 74.125.195.93, 74.125.195.136, 74.125.195.91, ...
    Connecting to dl.google.com (dl.google.com)|74.125.195.93|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 134784143 (129M) [application/x-gzip]
    Saving to: ‘go1.17.1.linux-amd64.tar.gz’
    
    go1.17.1.linux-amd6 100%[===================>] 128.54M   217MB/s    in 0.6s    
    
    2021-10-06 08:55:06 (217 MB/s) - ‘go1.17.1.linux-amd64.tar.gz’ saved [134784143/134784143]
    



```shell
!go version
```

    go version go1.17.1 linux/amd64


それではgo言語でzipの中身を覗いてみましょう。まずは以下でunzip.goファイルにプログラムを書き込みます。


```go
%%writefile unzip.go
package main

import (
    "archive/zip"
    "fmt"
    "log"
)

func main() {
    zfile, _ := zip.OpenReader("man.zip")
    defer zfile.Close()

    for _, f := range zfile.File {
        _, err := f.Open()
        if err != nil {
            fmt.Println(f.Method)
            log.Fatal(err)
        }
        fmt.Println(f.FileInfo().Name())
    }
}
```

    Overwriting unzip.go


早速実行！


```shell
!go run unzip.go
```

    93
    2021/10/06 08:55:46 zip: unsupported compression algorithm
    exit status 1


案の定、そんな圧縮方式知らぬと怒られました。しかし、メソッド番号が93であることが分かりました。実はよく調べたら、この番号は[ここ](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)に書いてありました。

そして、Go言語のzstdのモジュールは[github.com/klauspost/compress/zstd](https://pkg.go.dev/github.com/klauspost/compress/zstd)にありました。

完成したプログラムがこちら



```go
%%writefile unzip_fixed.go
package main

import (
    "archive/zip"
    "github.com/klauspost/compress/zstd"
    "fmt"
    "io"
    "log"
)

func main() {
    zfile, _ := zip.OpenReader("man.zip")
    defer zfile.Close()

	zfile.RegisterDecompressor(93, func(in io.Reader) io.ReadCloser {
        dec, _ := zstd.NewReader(in)
		return io.NopCloser(dec)
	})

    for _, f := range zfile.File {
        rc, err := f.Open()
        if err != nil {
            log.Fatal(err)
        }
        fmt.Println(f.FileInfo().Name())
        if !f.FileInfo().IsDir() {
            buf := make([]byte, f.UncompressedSize)
            n, _ := io.ReadFull(rc, buf)
            fmt.Println(n)
        }
    }
}
```

    Writing unzip_fixed.go


さっそく実行と行きたいところですが、外部モジュールを使っているので、その準備をば。


```shell
!go mod init unzip
```

    go: creating new go.mod: module unzip
    go: to add module requirements and sums:
    	go mod tidy



```shell
!go mod tidy
```

    go: finding module for package github.com/klauspost/compress/zstd
    go: downloading github.com/klauspost/compress v1.13.6
    go: found github.com/klauspost/compress/zstd in github.com/klauspost/compress v1.13.6


実行！


```shell
!go run unzip_fixed.go
```

    man.txt
    38134


ちゃんと解凍できました！

