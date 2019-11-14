# golang黑科技系列（一）万能的runime包

## 想获取当前程序所运行的系统类型及架构？So easy！
只需要直接引用runtime.GOOS或runtime.GOARCH。

注意返回值为string，根据官方文档，以GOOS/GOARCH列出所有可能值如下：

    aix/ppc64
    android/386
    android/amd64
    android/arm
    android/arm64
    darwin/386
    darwin/amd64
    darwin/arm
    darwin/arm64
    dragonfly/amd64
    freebsd/386
    freebsd/amd64
    freebsd/arm
    illumos/amd64
    js/wasm
    linux/386
    linux/amd64
    linux/arm
    linux/arm64
    linux/mips
    linux/mips64
    linux/mips64le
    linux/mipsle
    linux/ppc64
    linux/ppc64le
    linux/s390x
    nacl/386
    nacl/amd64p32
    nacl/arm
    netbsd/386
    netbsd/amd64
    netbsd/arm
    netbsd/arm64
    openbsd/386
    openbsd/amd64
    openbsd/arm
    openbsd/arm64
    plan9/386
    plan9/amd64
    plan9/arm
    solaris/amd64
    windows/386
    windows/amd64
    windows/arm
