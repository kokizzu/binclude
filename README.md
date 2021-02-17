*New Projects should use the official [embed](https://golang.org/pkg/embed/) package instead, which was added in go 1.16.*

# binclude

[![PkgGoDev](https://pkg.go.dev/badge/github.com/lu4p/binclude)](https://pkg.go.dev/github.com/lu4p/binclude)
[![Test](https://github.com/lu4p/binclude/workflows/Test/badge.svg)](https://github.com/lu4p/binclude/actions?query=workflow%3ATest)
[![Go Report Card](https://goreportcard.com/badge/github.com/lu4p/binclude)](https://goreportcard.com/report/github.com/lu4p/binclude)
[![codecov](https://codecov.io/gh/lu4p/binclude/branch/master/graph/badge.svg)](https://codecov.io/gh/lu4p/binclude)

binclude is a tool for including static files into Go binaries.
- focuses on ease of use
- the bincluded files add no more than the filesize to the binary
- uses go/ast for typesafe parsing
- each package can have its own `binclude.FileSystem`
- `binclude.FileSystem` implements the `http.FileSystem` interface
- `ioutil` like functions `FileSystem.ReadFile`, `FileSystem.ReadDir`
- include all files/ directories under a given path by calling `binclude.Include("./path")`
- include files based on a glob pattern `binclude.IncludeGlob("./path/*.txt")`
- add file paths from a textfile `binclude.IncludeFromFile("includefile.txt")`
- high test coverage
- supports execution of executables directly from a `binclude.FileSystem` via `binexec` (os/exec wrapper)
- optional compression of files with gzip `binclude -gzip`
- debug mode to read files from disk `binclude.Debug = true`

## Install
```
GO111MODULE=on go get -u github.com/lu4p/binclude/cmd/binclude
```
## Usage
```go
package main

//go:generate binclude

import (
	"io/ioutil"
	"log"
	"path/filepath"

	"github.com/lu4p/binclude"
)

var assetPath = binclude.Include("./assets") // include ./assets with all files and subdirectories

func main() {
	binclude.Include("file.txt") // include file.txt

	// like os.Open
	f, err := BinFS.Open("file.txt")
	if err != nil {
		log.Fatalln(err)
	}

	out, err := ioutil.ReadAll(f)
	if err != nil {
		log.Fatalln(err)
	}

	log.Println(string(out))

	// like ioutil.Readfile
	content, err := BinFS.ReadFile(filepath.Join(assetPath, "asset1.txt"))
	if err != nil {
		log.Fatalln(err)
	}

	log.Println(string(content))

	// like ioutil.ReadDir
	infos, err := BinFS.ReadDir(assetPath)
	if err != nil {
		log.Fatalln(err)
	}

	for _, info := range infos {
		log.Println(info.Name())
	}
}

```
To build use:
```
go generate
go build
```

A more detailed example can be found [here](https://github.com/lu4p/binclude/tree/master/example).

## Binary size
The resulting binary, with the included files can get quite large. 

You can reduce the final binary size by building without debug info (`go build -ldflags "-s -w"`) and compressing the resulting binary with [upx](https://upx.github.io/) (`upx binname`).

**Note:** If you don't need to access the compressed form of the files I would advise to just use [upx](https://upx.github.io/) and don't add seperate compression to the files. 

You can add compression to the included files with `-gzip`

**Note:** decompression is optional to allow for the scenario where you want to serve compressed files for a webapp directly.


## OS / Arch Specific Includes

binclude supports including files/binaries only on specific architectures and operating systems. binclude follows the same pattern as [Go's implicit Build Constraints](https://golang.org/pkg/go/build/#hdr-Build_Constraints). It will generate files for the specific platforms like `binclude_windows.go` which contains all windows specific files.

If a file's name, matches any of the following patterns: 
```
*_GOOS.go
*_GOARCH.go
*_GOOS_GOARCH.go
```
binclude will consider all files included by `binclude.Include` in this file as files which should only be included on a specific GOOS and/ or GOARCH.

For example, if you want to include a binary only on Windows you could have a file `static_windows.go` and reference the static file:
```go
package main

import "github.com/lu4p/binclude"

func bar() {
  binclude.Include("./windows-file.dll")
}
```

OS / Arch Specific Includes are used in the [binexec example](https://github.com/lu4p/binclude/tree/master/binexec/example).

**Note:** explicit build tags like `// +build debug` are not supported for including files conditionally.

## Advanced Usage
The generator can also be included into your package to allow for the code generator to run after all module dependencies are installed.
Without installing the binclude generator to the PATH seperatly.

main.go:
```go
// +build !gen

//go:generate go run -tags=gen .
package main

import (
	"github.com/lu4p/binclude"
)

func main() {
	binclude.Include("./assets")
}

```

main_gen.go:
```go
// +build gen

package main

import (
	"github.com/lu4p/binclude"
	"github.com/lu4p/binclude/bincludegen"
)

func main() {
	bincludegen.Generate(binclude.None, ".")
	// binclude.None == no compression 
	// binclude.Gzip == gzip compression
}
```

To build use:
```
go generate
go build
```

## Contributing
If you find a feature missing, which you consider a useful addition please open an issue. 

If you plan on implementing a new feature please open an issue so we can discuss it first (I don't want you to waste your time).

Pull requests are welcome, and will be timely reviewed and merged, please provide tests for your code, if you don't know how to test your code open a pull request and I will suggest the right testing method.
