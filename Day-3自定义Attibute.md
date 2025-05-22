✅***2025/5/5***更新
**[[#对Attitbute的疑问？]]**
实现一个自定义的Attribute流程与实现自定义Type的流程差不多
# 1.定义枚举类型系统（OldDialectEunms.td）
***主要用于表示和处理张量类型布局（Tensor Layout），要记得每定义一个功能就要注册一下，你知道应该在哪里添加**
## 1.1定义枚举值
```
def Old_LAYOUT_NCHW :I32EnumAttrCase<"NCHW",0>;
def Old_LAYOUT_NHWC :I32EnumAttrCase<"NHWC",1>;
```
- 定义了两种张量布局：
	 - `NCHW`（通道优先，对应整数值 `0`）
	 - `NHWC`（通道最后，对应整数值 `1`）
- `I32EnumAttrCase` 表示这是一个整数类型（32 位）的枚举值，字符串标签与整数值一一对应。
## 1.2创建枚举属性类型
```
def Old_Layout :I32EnumAttr<
    "Layout","Layout of tensor", //枚举名称和描述
    [Old_LAYOUT_NCHW,Old_LAYOUT_NHWC] //枚举值列表
>{
    let genSpecializedAttr = 0; //不生成专用属性类
    let cppNamespace = "::mlir::old"; //C++命名空间
}
```
## 1.3定义方言级别的属性包装器
```
def LLH_LayoutAttr :EnumAttr<OldDialect, Old_Layout, "Layout"> {
    let assemblyFormat = "`<` $value `>`";  // 语法格式：<NCHW> 或 <NHWC>
    
    let extraClassDeclaration = [{
        bool isChannelLast();  // 声明额外的方法
    }];
}
```
- `LLH_LayoutAttr` 是 `OldDialect` 方言中使用的布局属性。
- `assemblyFormat` 定义了属性在文本格式中的表示方式（例如 `<NCHW>`）。
- `extraClassDeclaration` 为生成的 C++ 类添加了一个方法声明 `isChannelLast()`，用于判断布局是否为通道最后（如 `NHWC`）。
## 1.4典型应用场景
- 在定义张量操作（如卷积、池化）时，使用 `LLH_LayoutAttr` 约束输入 / 输出张量的布局。
- 在转换过程中，通过 `isChannelLast()` 方法判断布局类型，从而生成不同的代码实现。
- 在 MLIR 模块的文本格式中，使用 `<NCHW>` 或 `<NHWC>` 明确指定张量布局。
# 2.定义属性系统（OldDialectAttrs.td）
## 2.1数据并行性属性
```
def Old_DataParallelism : Old_Attr<"DataParallelism", "DP", []> {
    let parameters = (ins "int64_t":$DP_nums);  // 参数：并行度数值
    let assemblyFormat = [{
        `<` `DP` `=` $DP_nums `>`  // 文本格式：<DP=4>
    }];
}
```
- 定义了一个名为 `DataParallelism` 的属性，缩写为 `DP`。
- 该属性接受一个 `int64_t` 类型的参数 `DP_nums`，表示数据并行度（如并行线程数）。
- 在文本格式中，该属性表示为 `<DP=4>`（假设并行度为 4）。
# 3.OldDialectAttrs实现
类似自定义Type实现
```
#include "/home/old/Old-project/OldDialect/include/Dialect/Old/OldDialectAttrs.h"
#include "/home/old/Old-project/OldDialect/include/Dialect/Old/OldDialectEunms.h"
#include "/home/old/Old-project/OldDialect/include/Dialect/Old/OldDialect.h"
//-------------------------------------------------------------------------------------------
#include "/home/old/Old-project/third_party/llvm-project/llvm/include/llvm/ADT/TypeSwitch.h"
#include "/home/old/Old-project/third_party/llvm-project/llvm/include/llvm/Support/LogicalResult.h"
#include "/home/old/Old-project/third_party/llvm-project/llvm/include/llvm/Support/raw_ostream.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/Dialect/Arith/IR/Arith.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/BuiltinTypeInterfaces.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/DialectImplementation.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/OpImplementation.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/Support/LLVM.h"
#define FIX
#define GET_ATTRDEF_CLASSES
#include "/home/old/Old-project/build/OldDialect/include/Dialect/Old/OldDialectAttrs.cpp.inc"
#include "/home/old/Old-project/build/OldDialect/include/Dialect/Old/OldDialectEunms.cpp.inc"

namespace mlir::old{
void OldDialect::registerAttrs(){
  llvm::outs()<<"register "<<getDialectNamespace()<<" Attr\n";

  addAttributes<
#define GET_ATTRDEF_LIST
#include "/home/old/Old-project/build/OldDialect/include/Dialect/Old/OldDialectAttrs.cpp.inc"
  >();
}
bool LayoutAttr::isChannelLast() {return getValue() == Layout::NHWC;}
}
#undef FIX

```
# 4.编写main部分
```void Day_3(){
    mlir::DialectRegistry registry;
    mlir::MLIRContext context(registry);
    auto dialect = context.getOrLoadDialect<mlir::old::OldDialect>();

    // Layout Eunms
  auto nchw = mlir::old::Layout::NCHW;
  llvm::outs() << "NCHW: " << mlir::old::stringifyEnum(nchw) << "\n";
  // LayoutAttr
  auto nchw_attr = mlir::old::LayoutAttr::get(&context, nchw);
  llvm::outs() << "NCHW LayoutAttribute :\t";
  nchw_attr.dump();
  // DataParallelismAttr
  auto dp_attr = mlir::old::DataParallelismAttr::get(&context, 2);
  llvm::outs() << "DataParallelism Attribute :\t";
  dp_attr.dump();
}

```

-------------------------------------------------------------------------
发生这个报错是不要慌，只需要在`#endif`后面回车一下，建立一个新的空行就行了
![[Pasted image 20250506101404.png]]
------------------------------------------------------------------------------------------------------------
# 对Attitbute的疑问？
attibute到底是什么，AI给出的回答是：是一种用于存储常量值或元数据的核心机制，与类型（Type）和操作（Operation）共同构成 MLIR 的三大基础组件
- Attribute 可存储各种常量数据，如整数、浮点数、字符串、数组等。
- 为操作（Operation）提供额外参数或配置信息。（这个看起来像是int a这么一个变量，或者是给一个函数传参的那个参数）
**不对劲**，类型（Type）和属性（attribute）的核心区别：
#### **类型（Type）**
- **作用**：定义值的结构、语义和约束，不存储具体数据。
- **特点**：不可变，编译期静态确定。
```
i32                  // 32位整数类型
tensor<4x8xf32>      // 4x8的浮点张量类型
!old.tensor<[1,2],f32,0>  // 自定义OldTensorType（shape=[1,2]，element_type=f32，device_id=0）
```
可以对应到C++中的数据类型或数据结构`（int、float32、String等）`
==**！！！确定数据的“框架”（如i32表示32位整数）**==
#### **属性（Attribute）**
- **作用**：**存储**具体的常量值或元数据。
- **特点**：不可变，可作为操作的参数或类型的参数。
```
42 : i32             // 整数属性（值为42，类型为i32）
"hello" : string     // 字符串属性
[1, 2, 3] : array    // 数组属性
{key = 42} : dict    // 字典属性
#old.device<0>       // 自定义Device属性（值为0）
```

他们之间的关系更像在C++**模板（Template）** 与**模板参数（Template Argument）** 的关系，我现在对它的理解就是C++中的右值，相同部分是均没有持久的和具体的内存地址，`Attribute`的数据只能作为元数据嵌入到IR中，不可被修改，更准确的类比应该是C++中的字面量。
==**！！！填充框架的“内容”（如42是i32类型下的具体值）**==
**！不同点：** 
- ### 作用域与声明周期
	- **右值**：C++ 右值的生命周期通常限于表达式（如临时对象在语句结束后销毁）。
	- **Attribute**：MLIR 中的 Attribute 的生命周期与 IR 绑定，只要 IR 存在，Attribute 就存在。
- ### 类型系统中的角色
	- **右值**：C++ 右值是表达式的值类别，与类型无关（如 `int` 类型的右值 `42`）。
	- **Attribute**：MLIR 中的 Attribute 本身就是一种类型（如 `IntegerAttr`、`StringAttr`），且每个 Attribute 必须关联一个 MLIR 类型（如 `42 : i32`）。
- ### 用途不同
	- **右值**：C++ 右值主要用于表达式计算或移动语义。
	- **Attribute**：MLIR 中的 Attribute 主要用于**携带常量值**、**参数化类型**或**描述操作的元数据**




