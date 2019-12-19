# Makefile与编译脚本分析

## 代码生成
makefile在make all之前会先generated_files去进行代码生成，所以首先要理解代码生成的原理，然后才可以很好的知道
这个过程都干啥了.
```
all: generated_files
	hack/make-rules/build.sh $(WHAT)
```

首先，为什么需要代码生成？答案是“懒”。写过代码的都知道很多时候有大量结构重复的代码需要去写，劳心劳力还没什么技术含量，所以为了解决这个问题代码生成它来了。

代码生成的场景有很多如：

* protobuf 根据一个协议字段配置文件生成客户端和服务端的.go代码
* IDE中的自动测试用例和接口实现函数代码生成
* 一些web框架自动生成RESTFUL接口代码
* operator脚手架工具生成k8s controller代码等

在kubernetes中主要生成代码有这些：

* deep-copy generator, kubernetes中的对象都需要实现该方法，每个对象都自己手动去写很累，因为kubernetes需要把期望状态与实际状态进行比对，所以需要把对象深拷贝一份再比对
* defaulter generator 不重要，可以用来填充些静态内容的文件
* go-to-protobuf generator, 组件之间都是通过protobuf进行通信

### 代码生成原理
所以我们的目的就是根据源代码再生成一些源代码，那问题就分成三步走：

1. 解析我们写的源码，提取我们所需要的内容，如包名，结构体名，等
2. 渲染模板文件
3. 生成源码文件

下面用个简单的例子来帮助理解这一过程。

> 安装stringer

stringer可以帮助枚举类型自动生成String()方法
```
go install golang.org/x/tools/cmd/stringer
```

> 编码

```
cd $GOPATH/src
mkdir gen && cd gen && touch main.go
```
我们在main.go里输入以下内容：

```
package main

import "fmt"

//go:generate stringer -type=Pill
type Pill int

const (
	Placebo Pill = iota
	Aspirin
	Ibuprofen
	Paracetamol
	Acetaminophen = Paracetamol
)

func main() {
	fmt.Println(Placebo.String())
}
```
注意现在只有一个main.go 且里面并没有实现String()方法，String方法输出药的名字，而main里面却调用了，那自然便宜不过：
```
# github.com/fanux/gen
./main.go:17:21: Placebo.String undefined (type Pill has no field or method String)
```

如果我们自己要去实现String()方法枯燥无味且还需要对药进行判断，药越多switch case越多，假设有10000种药，那么我们直接就崩溃了

> 代码生成

因为有这个注释，go generate时就会调用stringer工具进行代码生成
```
//go:generate stringer -type=Pill
```
```
# go generate && go build && ls
gen            go.mod         main.go        pill_string.go
```
可以看到就生成了一个pill_string.go文件，里面实现了我们想要的String方法：
```
func _() {
	// An "invalid array index" compiler error signifies that the constant values have changed.
	// Re-run the stringer command to generate them again.
	var x [1]struct{}
	_ = x[Placebo-0]
	_ = x[Aspirin-1]
	_ = x[Ibuprofen-2]
	_ = x[Paracetamol-3]
}

const _Pill_name = "PlaceboAspirinIbuprofenParacetamol"

var _Pill_index = [...]uint8{0, 7, 14, 23, 34}

func (i Pill) String() string {
	if i < 0 || i >= Pill(len(_Pill_index)-1) {
		return "Pill(" + strconv.FormatInt(int64(i), 10) + ")"
	}
	return _Pill_name[_Pill_index[i]:_Pill_index[i+1]]
}
```

函数实现成什么样子不用纠结，知道生成了我们需要的代码就行，运行一下：
```
# ./gen
Placebo
```

> 原理

stringer是如何做到的，很简单，让我们一起去看一下其源码，其中最重要的就是go/ast语法树解析和go/parser解析库的运用

首先我们需要生成代码肯定需要知道包名是什么：

packages库的LOAD函数可以帮助我们做到
```
golang.org/x/tools/go/packages
pkgs, err := packages.Load(cfg, patterns...)
```

并且获取到package下面的所有文件信息：
```
for i, file := range pkg.Syntax {
	g.pkg.files[i] = &File{
		file:        file,
		pkg:         g.pkg,
		trimPrefix:  g.trimPrefix,
		lineComment: g.lineComment,
	}
}
```
