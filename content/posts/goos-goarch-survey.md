---
title: "GOOS/GOARCH combos on macOS"
date: 2019-01-13T21:11:10+11:00
draft: true
---

The following is a survey of GOOS/GOARCH support on my Mac, using [src/go/build/syslist.go@7cbfa5](https://github.com/golang/go/blob/7cbfa55b5d17c8deaecff05e4221f828467cfa97/src/go/build/syslist.go) as the source of combinations to try. The following table shows combinations that built successfully (✅) and those that failed (❌). The rest, denoted by ‘—’, were reported by Go as unsupported combinations, e.g.:

```bash
$ GOOS=js GOARCH=386 go build
cmd/go: unsupported GOOS/GOARCH pair js/386
```

{{< bootstrap-table "table table-striped table-condensed table-bordered" >}}
| <span style="font-size: smaller">GOARCH&nbsp;→<br/>GOOS<br/>↓</span> | 386  | amd64  | <span style="font-size: smaller">amd64-p32</span> | arm  | arm64  | <span style="font-size: smaller">mips/le<br/>mips64/le<br/>ppc64/le<br/>s390x</span> | riscv64 | wasm |
| :-------- | :--: | :----: | :------: | :--: | :----: | :---------------------:               | :------: | :--: |
| android   |  ❌¹ |   ❌¹ |    —     |  ❌¹ |   ❌¹ |           —                           |    —     |  —   |
| darwin    |  ✅  |   ✅  |    —     |  ❌² |   ❌² |           —                           |    —     |  —   |
| dragonfly |  —   |   ✅   |    —     |      |   —   |           —                            |    —     |  —   |
| freebsd   |  ✅  |   ✅  |    —     |  ✅  |   —    |           —                           |    —     |  —   |
| js        |  —   |   —    |    —     |  —   |   —    |             —                         |    —     |  ✅ |
| linux     |  ✅  |   ✅  |    —     |  ✅  |   ✅  |           ✅                          |    ❌³  |  —   |
| nacl      |  ✅  |   —    |    ✅   |  ✅  |   —    |           —                           |    —     |  —   |
| netbsd    |  ✅  |   ✅  |    —     |  ✅  |   —    |           —                           |    —     |  —   |
| openbsd   |  ✅  |   ✅  |    —     |  ✅  |   —    |           —                           |    —     |  —   |
| plan9     |  ✅  |   ✅  |    —     |  ✅  |   —    |           —                           |    —     |  —   |
| solaris   |  —   |   ✅   |    —     |  —   |   —   |          —                             |    —     |  —   |
| windows   |  ✅  |   ✅  |    —     |  —   |   —    |           —                           |    —     |  —   |
{{< /bootstrap-table >}}

¹ `ld: unknown option: -z` <br/>
² `ld: symbol(s) not found for architecture x86_64` (_main) <br/>
³ `compile: unknown architecture "riscv64"`

The following OSes and architectures had no support at all:

- OSes: `aix` `hurd` `zos`
- architectures: `arm(64)?be` `mips64p32(le)?` `ppc` `riscv` `s390` `sparc(64)?`

#### Config

```bash
$ uname -a
Darwin … 18.2.0 Darwin Kernel Version 18.2.0: Mon Nov 12 20:24:46 PST 2018; root:xnu-4903.231.4~2/RELEASE_X86_64 x86_64 i386 MacBookPro10,1 Darwin
$ go version
go version go1.11.4 darwin/amd64
```

## Detailed output for failed builds

```bash
$ cat hello.go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello, playground")
}
$ GOOS=android GOARCH=386 go build
# play/hello
/usr/local/Cellar/go/1.11.4/libexec/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: unknown option: -z
clang: error: linker command failed with exit code 1 (use -v to see invocation)

$ GOOS=android GOARCH=amd64 go build
# play/hello
/usr/local/Cellar/go/1.11.4/libexec/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: unknown option: -z
clang: error: linker command failed with exit code 1 (use -v to see invocation)

$ GOOS=android GOARCH=arm go build
# play/hello
/usr/local/Cellar/go/1.11.4/libexec/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: unknown option: -z
clang: error: linker command failed with exit code 1 (use -v to see invocation)

$ GOOS=android GOARCH=arm64 go build
# play/hello
/usr/local/Cellar/go/1.11.4/libexec/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: unknown option: -z
clang: error: linker command failed with exit code 1 (use -v to see invocation)

$ GOOS=darwin GOARCH=arm go build
# play/hello
/usr/local/Cellar/go/1.11.4/libexec/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: warning: ignoring file /var/folders/x_/03w67k_92jq2xn2zbgm2n9k40000gn/T/go-link-440104962/go.o, file was built for armv7 which is not the architecture being linked (x86_64): /var/folders/x_/03w67k_92jq2xn2zbgm2n9k40000gn/T/go-link-440104962/go.o
Undefined symbols for architecture x86_64:
  "_main", referenced from:
     implicit entry/start for main executable
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)

$ GOOS=darwin GOARCH=arm64 go build
# play/hello
/usr/local/Cellar/go/1.11.4/libexec/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: warning: ignoring file /var/folders/x_/03w67k_92jq2xn2zbgm2n9k40000gn/T/go-link-419067768/go.o, file was built for unsupported file format ( 0xCF 0xFA 0xED 0xFE 0x0C 0x00 0x00 0x01 0x00 0x00 0x00 0x00 0x01 0x00 0x00 0x00 ) which is not the architecture being linked (x86_64): /var/folders/x_/03w67k_92jq2xn2zbgm2n9k40000gn/T/go-link-419067768/go.o
Undefined symbols for architecture x86_64:
  "_main", referenced from:
     implicit entry/start for main executable
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)

$ GOOS=linux GOARCH=riscv64 go build
go tool compile: exit status 2
compile: unknown architecture "riscv64"
go tool compile: exit status 2
compile: unknown architecture "riscv64"
```

The last one is not a copy-paste mistake. The linux/riscv64 combination produces duplicate error messages, usually double, but on one run it printed the same error eight times.
