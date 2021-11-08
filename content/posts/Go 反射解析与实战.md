---
title: "Go 反射解析与实战"
author: "小贺"
date: 2021-08-30T17:10:12+08:00
tags: ["goland", "反射"]
---
## 前言
日常写业务时，我们很少会用到反射，导致大部分人对 go 的反射还比较陌生。
虽然并不推荐在业务代码中写反射代码，但是了解它，能够让我们更好的去理解许多框架的逻辑，以及能够让自己具备有初步实现一个通用第三方 SDK 的能力。

#### 什么是反射
> 在计算机学中，反射是指计算机程序在运行时（runtime）可以访问、检测和修改它本身状态或行为的一种能力。用比喻来说，反射就是程序在运行的时候能够“观察”并且修改自己的行为。

来自维基百科

在 Goland 中，其实就是编译时是 interface{}，不知道其具体的类型，要在运行中反射获取类型、更新检查他们的值、

执行它们的方法，

#### 反射能够干什么
反射是元编程的一个关键策略
- java spring
- goland json 序列化与反序列化
- go orm 框架
- ...

#### 反射的缺点
- 性能
- 可读性
- go 作为静态语言，编译时能检查出不少问题，但反射跳过这检查，可能在运行中 panic

## Goland 反射解析

要搞清楚反射，得先简要了解一下 interface

### Interface
#### 定义

接口有两种定义：

- eface: 空接口定义, 包含具体类型与数据

- iface:  非空接口定义（实现了方法的接口），

```go
// 位于 src/runtime/runtime2.go
type eface struct {
	_type *_type					// 具体的类型
	data  unsafe.Pointer			// 数据
}
type iface struct {
	tab  *itab					// 指向itab 的指针
	data unsafe.Pointer			// 数据
}

```

_type 就是所有类型最原始的元信息

```go
type _type struct {
	size       uintptr 		// 类型占用内存大小
	ptrdata    uintptr
	hash       uint32
	tflag      	tflag   	  // 标记位，主要用于反射
	align      uint8   		// 对齐字节信息
	fieldAlign 	uint8   
	kind       uint8   		// 基础类型枚举值,  26 个基础类型, int，ptr，struct，interface 
	equal func(unsafe.Pointer, unsafe.Pointer) bool // 比较两个形参对应对象的类型是否相等
  ...
}
```

iface 中 itab 相对复杂，存放的是类型、方法等信息

```go
// 位于 src/runtime/iface.go
type itab struct {
    inter  *interfacetype	// 包装了一层*_type，代表接口类型，go 中 中基础类型slice，chan 的类型 都有定义，并且都是包装了一层*_type
    _type  *_type			// 等同于 eface 中的 *_type, 具体对象的类型
    link   *itab
    bad    int32
    inhash int32      		// has this itab been added to hash?
    fun    [1]uintptr 		// 这里存的是指向第一个方法的指针，其他方法在这个地址后按字典序存储，偏移量即可
}
```

#### 各种基础类型的类型
```go
// 位于 src/runtime/type.go
type interfacetype struct {
	typ     _type
	pkgpath name		// 包路径
	mhdr    []imethod	// 方法
}

type arraytype struct {
    typ   _type
    elem  *_type	// array 上具体元素的类型
    slice *_type
    len   uintptr
}

type chantype struct {
    typ  _type
    elem *_type		// channel 上具体元素的类型
    dir  uintptr
}
// ...
```

#### Interface 总结

一个具体的对象转换成 interface后，**类型信息与数据会分别存在在 _type，data 两个指针中，不会丢失信息**（所以后面反射能够还原出实际对象），如果这个具体的对象还实现了一些接口的函数，方法列表保存在 fun 指向的地址。

使用 interface 还有一个常见的坑： 判断一个 interface 是不是 nil，只要类型信息不为空，则这个 interface != nil

### 反射
此处分析的代码都在 src/reflect 包下

反射最基本的两个方法
```go
TypeOf(i interface{}) Type		// 对应着 interface 结构体中 *_type
ValueOf(i interface{}) Value	// 结合了data指针与 *_type 类型信息
```
在继续解析代码前，我们可以先思考以下几个问题
- 为什么 Value 有 Interface() (i interface{}) 而 Type 没有
- Kind 与 Type 的区别
- 我们期望从 struct 通过反射获取到什么信息

#### Type

反射包下的 Type 是一个接口，其倒数第两个方法 common()  rtype其实就是上文中的 _type

```go
// 位于 src/reflect/type.go 下
type Type interface {
	Align() int
	FieldAlign() int
	Method(int) Method										// 返回类型方法集里的第 `i` (传入的参数)个方法
	MethodByName(string) (Method, bool) 	// 通过名称获取方法
	NumMethod() int												// 方法个数
	Name() string													// 类型名字
	Size() uintptr
	String() string
	Kind() Kind														// 所属基础类型
	Implements(u Type) bool
	AssignableTo(u Type) bool							
	ConvertibleTo(u Type) bool						// 能否转换 u
	Comparable() bool
	Bits() int
	ChanDir() ChanDir											// channel 类型的方向
	IsVariadic() bool
	Elem() Type														// 返回内部子元素类型，只能由类型 Array, Chan, Map, Ptr, or Slice 调用
	Field(i int) StructField							// 获取结构体中的 字段
	FieldByIndex(index []int) StructField
	FieldByName(name string) (StructField, bool)
	FieldByNameFunc(match func(string) bool) (StructField, bool)
	In(i int) Type
	Key() Type												// 返回 map 的 key 类型，只能由类型 map 调用
	Len() int
	NumField() int
	NumIn() int									// 入参个数
	NumOut() int								// 出参个数

	Out(i int) Type

	common() *rtype							// 与 interface 结构中 _type 一一对应
	uncommon() *uncommonType		// 与 interface 结构中 itab 所包含内容对应，所具备的方法，包名
}
```

Type 的方法比较多, 个人认为下面几个比较**常用**，可以仔细去看一下具体代码
- ```
  MethodByName(string) (Method, bool)  通过名称获取方法
  ```

- ```
  FieldByName(name string) (StructField, bool) 根据名字获取struct中的字段信息，其中常用到字段信息包括字段名，类型，tag
  ```

- ```
  Elem() Type 如果这个 type 是 Array, Chan, Map, Ptr, or Slice，返回其指向的具体元素的类型
  ```

- ```
  AssignableTo(u Type) bool 判断两个类型的值能够直接赋值（类型相等，或者其中一个类型没有定义）
  ```

- ```
  ConvertibleTo(u Type) bool 判断两个类型 是否能够强制转换, 如 Int 之间，int 与 float，string 与 int32 能够转换。
  ```



其实 Type 接口的之所以具备这么多方法，是为了使用便利与通用性。实际上很多都是特异化的，例如常用到的 Field(i int), 只有基础类型 为 Struct 的才能使用，Elem() 只能由类型 Array, Chan, Map, Ptr, or Slice 调用, 其他的类型调用会直接 panic。

下面的代码展示了struct 的解析，我们以及能够 struct 字段里面 字段名、类型、tag 等信息

```go
// 位于/reflect/type.go 下 struct 类型的结构
type structType struct {
	rtype
	pkgPath name
	fields  []structField // sorted by offset
}

func (t *rtype) FieldByName(name string) (StructField, bool) {
	if t.Kind() != Struct {
		panic("reflect: FieldByName of non-struct type " + t.String())
	}
	tt := (*structType)(unsafe.Pointer(t))
	return tt.FieldByName(name)
}

// A StructField describes a single field in a struct.
type StructField struct {
	Name string					// 字段名
	PkgPath string			// 包路径

	Type      Type      // 字段类型
	Tag       StructTag // 字段 tag
	Offset    uintptr   // offset within struct, in bytes
	Index     []int     // index sequence for Type.FieldByIndex
	Anonymous bool      // is an embedded field
}
```

**总结**

TypeOf(i interface) 返回一个接口，通过这些接口的方法能够获取所有类型信息

#### Value

Value 是一个结构体， 其包含类型与数据信息

```go
type Value struct {
	typ *rtype						// 类型
	ptr unsafe.Pointer		// 实际数据
	flag					 				// 标记位
}

```

Value 常用的方法

- ```go
  Type() Type  返回具体类型
  ```

- ```go
  Interface() (i interface{})  value 有类型又有数据，可以直接转化成 interface
  ```

- ```go
   MethodByName(name string) Value 返回方法
  ```

- ```go
  FieldByName(name string) Value	根据字段名找出字段对应的值，在 Type 有类似的方法，不过 Type 返回的是字段类型信息
  ```

- ```
  Indirect(v Value) Value	 这个 value 如果是指针，返回它指向的 value，不是返回本身，实际使用 Elem() 实现
  ```

- ```go
  Elem() Value 返回指向的对象,并且会标记为 addr， 返回的 value 能用Addr() 获取地址 
  ```

- ```
  CanSet() bool 判断 value 里的数据能否被改变，满足可寻址的条件（CanAddr），如果是字段是对外暴露的（字段名大写）
  ```

- ```
  Convert(t Type) Value 改变 value 的具体类型，可以利用上面Type中提到的 ConvertibleTo方法 来判断是否可以转换，如int -> int64
  ```

- ```
  Call(in []Value) []Value	调用 Call Value Kind必须是函数，用 in 作为参数，返回 []Value, 反射执行方法
  ```

- 取出 value 中的值转换成具体类型

  ```go
  Float() float64	取出 float得值
  ```
  ```go
  Int() int64	 取出 int64 的值
  ```
  ...

- 改变 value 的值
  ```
  Set(x Value) 把 value 里的值设置成 x， value 满足CanSet()
  ```
  ```
  SetInt(x int64) 作用类似
  ```

  ```text
  SetLen(n int) 切片设置长度，不是切片 panic
  ```

**总结**

ValueOf(i interface) 返回一个结构体，这个结构体包含 <类型，数据> 信息， 并且提供了许多方法，来获取，修改里面的信息（类型，数据）

## 代码实战

#### 能否开发一个 copy 函数完成对象之间的复制函数

web 开发PO 转化成 DTO 的代码必不可少，这些代码比较重复枯燥，写个方法提升下效率

```go
func Copy(dest, src interface{}) error {
	// dest 预期是指向对象的ptr，Go 中是值传递，不是 ptr 的话，修改会无效
	// Indirect 如果这个值是指针，会返回指向的值，其中 flag 会加上 CanAddr 的标记
	destValue := reflect.Indirect(reflect.ValueOf(dest))
	if !destValue.CanAddr() {
		return fmt.Errorf("dest type is not ptr")
	}
	
	// 反射 src 得到 src 的类型 与 对象值
	srcValue := reflect.ValueOf(src)
	srcType := reflect.TypeOf(src)
	if srcType.Kind() == reflect.Ptr {
		srcType = srcType.Elem()
	}
	// 遍历 src类型字段
	for i := 0; i < srcType.NumField(); i ++ {
		fieldSrc := srcType.Field(i)
		// 根据 src 的字段名 在 dest 的结构中查找
		fieldDest, exist := destValue.Type().FieldByName(fieldSrc.Name)
		// dest 不存在同名的字段跳过
		if !exist {
			continue
		}
		// 判断 src 字段类型能否转化成 dest 字段类型
		if ok := fieldSrc.Type.ConvertibleTo(fieldDest.Type); ok {
			// 获取 src value 对象中具体的字段值，并且转换成 dest 字段的类型
			convertValue := srcValue.FieldByName(fieldSrc.Name).Convert(fieldDest.Type)
			// 设置 dest value 对象的相应字段的值
			destValue.FieldByName(fieldSrc.Name).Set(convertValue)
		}
	}
	return nil
}
```

上面就是一个对象 Copy 的主要代码，依靠字段名复制，主要问题有两个
- 完全依靠字段名匹配，字段名不一致且不好改了，能否完成复制
- 字段类型转换完全依靠内置的ConvertibleTo(u Type) bool 函数，如果我想完成 int64 -> time.time 的复制该怎么做

当然这个代码已经有比较好的实现了，见 https://github.com/jinzhu/copier, 目前 2.2 k star, 如此简单的思想，但又如此实用，是不是对反射的代码更有兴趣了呢。


#### orm 中需要对数据库表与实体model做一个映射，如何实现？
```go
// Schema 解析 model 得到的元数据
type Schema struct {
	Model      interface{}			// 实体对象
	Name       string			 	// 实体的名字，作为表名
	Fields     []*Field				// 字段列表，转化成 sql 中用到的信息
	FieldNames []string				// 字段名列表，这里用实体字段名作为 sql 中字段名
	fieldMap   map[string]*Field	// map[字段名] 字段
}

// Field sql 表中字段
type Field struct {
	Name string		// sql 字段名
	Type string		// sql 类型 int bigint..
	Tag  string		// 额外信息，例如 primary key
}

// model 作为与数据库表一一对应的实体，dialect 代表一种类型转换规则，例如 go -> mysql 中 string -> varchar
// 而 go -> sqlite中， string -> text
func Parse(model interface{}, dialect dialect.Dialect) *Schema {
	// 获取 model 的实际类型
	modelType := reflect.Indirect(reflect.ValueOf(model)).Type()
	schema := &Schema{
		Model:    model,
		Name:     modelType.Name(),		// 类型信息中 有实体名字
		fieldMap: map[string]*Field{},
	}
	// 遍历实体中每个字段
	for i := 0; i < modelType.NumField(); i++ {
		// 字段信息
		structField := modelType.Field(i)
		// 字段必须是不是匿名的和对外暴露的
		if !structField.Anonymous && ast.IsExported(structField.Name) {
			// 根据 go 对象 构建 sql 表中字段
			field := &Field{
				Name: structField.Name,		// 字段的名字
				Type: dialect.DataTypeOf(reflect.Indirect(reflect.New(structField.Type))),
				// 这里把 go 中的类型转换成 sql 中的类型，dialect 有多个实现，例如 sqlite， mysql
			}
			// 从 structFiled 获取需要的 tag 信息
			if v, ok := structField.Tag.Lookup("orm"); ok {
				field.Tag = v
			}
			// 存入 schema 中
			schema.Fields = append(schema.Fields, field)
			schema.FieldNames = append(schema.FieldNames, field.Name)
			schema.fieldMap[field.Name] = field
		}
	}
	return schema
}
```

上面的代码就完成了一个 model 对象对数据库表的映射，主要有：

- 表名 -> 数据库表名
- 字段名 -> 数据库表字段名
- 字段类型 -> 数据库表字段的类型
- tag -> 字段的一些额外信息

这里的功能就是 gorm 框架 https://github.com/go-gorm/gorm/blob/master/schema/schema.go 的核心逻辑，对自己写一个 orm 是不是更有信心一点了呢。

#### 课后作业

- reflect DeepEqual 方法 源代码

- json Marshal 方法 源代码
- govalidator 自己实现一个参数校验的方法，

---

以上就是全部内容了

最后再来一个问题， 2021 年底，go 会在 1.18 版本中添加泛型，你们能说一下 interface 与 泛型的区别吗