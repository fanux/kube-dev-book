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
注意现在只有一个main.go 且里面并没有实现String()方法，String方法输出药的名字，而main里面却调用了，那自然编译不过：
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

ast库中的Inspect函数可以深度优先的去遍历语法树，树的每个节点会传递给file.genDecl函数去处理
```
ast.Inspect(file.file, file.genDecl)
```

遍历声明的元素，每个元素都是一个ValueSpec:
```
for _, spec := range decl.Specs {
	vspec := spec.(*ast.ValueSpec) 
```
最终遍历得到枚举的名字并保存下来，这也是生成代码所需要的
```
for _, name := range vspec.Names {
    ...
	v := Value{
		originalName: name.Name, // 我们想要的
		value:        u64,
		signed:       info&types.IsUnsigned == 0,
		str:          value.String(),
	}
    ...
}
```

最后用一些模板把代码渲染出来：
```
const stringOneRun = `func (i %[1]s) String() string {
	if %[3]si >= %[1]s(len(_%[1]s_index)-1) {
		return "%[1]s(" + strconv.FormatInt(int64(i), 10) + ")"
	}
	return _%[1]s_name[_%[1]s_index[i]:_%[1]s_index[i+1]]
}`

const stringOneRunWithOffset = `func (i %[1]s) String() string {
	i -= %[2]s
	if %[4]si >= %[1]s(len(_%[1]s_index)-1) {
		return "%[1]s(" + strconv.FormatInt(int64(i + %[2]s), 10) + ")"
	}
	return _%[1]s_name[_%[1]s_index[i] : _%[1]s_index[i+1]]
}
```
所以就没什么神奇的了

如果觉得上面还是不太懂，那么看个更简单的例子：
```
func main() {
    // 这是我们需要进行语法树分析的代码
	src := `
package p
const c = 1.0
var X = f(3.14)*2 + c
`

    // 创建一个AST
	fset := token.NewFileSet() 
	f, err := parser.ParseFile(fset, "src.go", src, 0)
	if err != nil {
		panic(err)
	}

    // 遍历语法树节点
	ast.Inspect(f, func(n ast.Node) bool {
		var s string
		switch x := n.(type) {
		case *ast.BasicLit:
			s = x.Value
		case *ast.Ident:
			s = x.Name
		}
		if s != "" {
			fmt.Printf("%s:\t%s\n", fset.Position(n.Pos()), s)
		}
		return true
	})

}
```
会输出：
```
src.go:2:9:     p
src.go:3:7:     c
src.go:3:11:    1.0
src.go:4:5:     X
src.go:4:9:     f
src.go:4:11:    3.14
src.go:4:17:    2
src.go:4:21:    c
```
可以看到变量名，变量值都被输出了，然后我们就可以拿着这些结果去渲染代码了

我们还可以输出完整的语法树看一下：
```
    src := `
package main
func main() {
	println("Hello, World!")
}
`
	fset := token.NewFileSet() 
	f, err := parser.ParseFile(fset, "", src, 0)
	if err != nil {
		panic(err)
	}

	ast.Print(fset, f)
```
```
     0  *ast.File {
     1  .  Package: 2:1
     2  .  Name: *ast.Ident {
     3  .  .  NamePos: 2:9
     4  .  .  Name: "main"
     5  .  }
     6  .  Decls: []ast.Decl (len = 1) {
     7  .  .  0: *ast.FuncDecl {
     8  .  .  .  Name: *ast.Ident {
     9  .  .  .  .  NamePos: 3:6
    10  .  .  .  .  Name: "main"
    11  .  .  .  .  Obj: *ast.Object {
    12  .  .  .  .  .  Kind: func
    13  .  .  .  .  .  Name: "main"
    14  .  .  .  .  .  Decl: *(obj @ 7)
    15  .  .  .  .  }
    16  .  .  .  }
    17  .  .  .  Type: *ast.FuncType {
    18  .  .  .  .  Func: 3:1
    19  .  .  .  .  Params: *ast.FieldList {
    20  .  .  .  .  .  Opening: 3:10
    21  .  .  .  .  .  Closing: 3:11
    22  .  .  .  .  }
    23  .  .  .  }
    24  .  .  .  Body: *ast.BlockStmt {
    25  .  .  .  .  Lbrace: 3:13
    26  .  .  .  .  List: []ast.Stmt (len = 1) {
    27  .  .  .  .  .  0: *ast.ExprStmt {
    28  .  .  .  .  .  .  X: *ast.CallExpr {
    29  .  .  .  .  .  .  .  Fun: *ast.Ident {
    30  .  .  .  .  .  .  .  .  NamePos: 4:2
    31  .  .  .  .  .  .  .  .  Name: "println"
    32  .  .  .  .  .  .  .  }
    33  .  .  .  .  .  .  .  Lparen: 4:9
    34  .  .  .  .  .  .  .  Args: []ast.Expr (len = 1) {
    35  .  .  .  .  .  .  .  .  0: *ast.BasicLit {
    36  .  .  .  .  .  .  .  .  .  ValuePos: 4:10
    37  .  .  .  .  .  .  .  .  .  Kind: STRING
    38  .  .  .  .  .  .  .  .  .  Value: "\"Hello, World!\""
    39  .  .  .  .  .  .  .  .  }
    40  .  .  .  .  .  .  .  }
    41  .  .  .  .  .  .  .  Ellipsis: -
    42  .  .  .  .  .  .  .  Rparen: 4:25
    43  .  .  .  .  .  .  }
    44  .  .  .  .  .  }
    45  .  .  .  .  }
    46  .  .  .  .  Rbrace: 5:1
    47  .  .  .  }
    48  .  .  }
    49  .  }
    50  .  Scope: *ast.Scope {
    51  .  .  Objects: map[string]*ast.Object (len = 1) {
    52  .  .  .  "main": *(obj @ 11)
    53  .  .  }
    54  .  }
    55  .  Unresolved: []*ast.Ident (len = 1) {
    56  .  .  0: *(obj @ 29)
    57  .  }
    58  }
```
想拿啥都能拿到了

### kubernetes中的代码生成

kubernetes/gengo 项目专门用来做代码生成的，有了上面基础我就直接说结果了

咱打开kubernetes源码, `pkg/apis/core/types.go` 里有Pod结构体的定义
```
// Pod is a collection of containers, used as either input (create, update) or as output (list, get).
type Pod struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Spec defines the behavior of a pod.
	// +optional
	Spec PodSpec

	// Status represents the current information about a pod. This data may not be up
	// to date.
	// +optional
	Status PodStatus
}
```
注意同目录下`zz_generated.deepcopy.go` 文件就是生成的，为Pod生成深拷贝方法
这里面生成会有很多细节的处理，如链表的处理，指针的处理等，细节部分有兴趣可以[参考](https://github.com/kubernetes/gengo/blob/master/examples/deepcopy-gen/generators/deepcopy.go) 此处代码
```
func (in *Pod) DeepCopy() *Pod 
func (in *Pod) DeepCopyObject() runtime.Object 
```

  如此是不是学到了什么，比如我们在自定义资源时是不是可以用同样的方式来减少我们的工作量,当然现在如kubebuilder operator framework都帮我们这样做了，我们就不需要自己去实现代码生成器了。但是学会了AST我们自己写重复的业务逻辑时就会非常有用
