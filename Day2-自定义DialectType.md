# 当前出现的BUG
- 如下报错
![[Pasted image 20250503004601.png]]
目前来看问题出现在如下部分
```
！位于OldDialect.td中，当时设置参数为0
  // 使用MLIR默认的类型解析输出.
  let useDefaultTypePrinterParser = 1;
！位于OldDialect.cpp文件中，当时并未初始化registerType（）函数
  void OldDialect::initialize(){
        llvm::outs()<<"initialize: "<<getDialectNamespace()<<"\n";
        registerType();
    }
```
并且还发现红框中的宏有写错的情况
![[Pasted image 20250503005549.png]]
# 正式内容
没想到一个小BUG居然卡了我将近三天的时间，而且问题居然很简单就很难绷了，[时光丶人爱](https://space.bilibili.com/15558499) <-观看这个up的mlir教学视频，[MLIR-Tutorial](https://github.com/violetDelia/MLIR-Tutorial)<-建议clone下来这个项目作为对照和对比，这个教学视频有一个小问题就是有些文件修改内容不会直接告诉你，需要自己核对部分文件内容。**当前文章主要内容是如何实现自定义一个DialectType！！！**
## 首先是编写OldDialectTypes.td文件
```
#ifndef DIALECTBASE_NORTH_STAR_TYPES_TD
#define DIALECTBASE_NORTH_STAR_TYPES_TD

//include "mlir/IR/Utils.td"
include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/DialectBase.td"
include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/Traits.td"
include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/AttrTypeBase.td"
include "/home/old/Old-project/OldDialect/include/Dialect/Old/OldDialect.td"
include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/BuiltinTypeInterfaces.td"

//===----------------------------------------------------------------------===//
// Dialect definitions
//===----------------------------------------------------------------------===//
class Old_Type<string name, string typeMnemonic, list<Trait> traits = [],
                   string baseCppClass = "::mlir::Type">
    : TypeDef<OldDialect, name, traits, baseCppClass> {
  let mnemonic = typeMnemonic;
  let typeName =  dialect.name # "." # typeMnemonic;
}

def Old_TensorType:Old_Type<"OldTensor","old_tensor",[]>{

  let summary="Old Tensor Type";

  let description="Old Tensor Type";

  let parameters = (ins
    ArrayRefParameter<"int64_t","mlir::ArrayRef<int64_t>">:$shape,
    "Type":$elementType,
    "int64_t":$device_id
  );

  let genStorageClass =1;

  let hasStorageCustomConstructor =0;

  let builders=[
    TypeBuilder<(ins 
        "::mlir::ArrayRef<int64_t>":$shape,
        "::mlir::Type":$elementType
        //"int64_t":$device_id
        ),
        [{
      return $_get(elementType.getContext(), shape, elementType, 0);
    }]>
  ];


  let hasCustomAssemblyFormat=1;
  //let assemblyFormat = "`<`$shape`,`$elementType`,`$device_id`>`";

  let skipDefaultBuilders = 0;

  // 是否生成类型检验的函数声明
  let genVerifyDecl = 1;

  
}

#endif // DIALECTBASE_TD

```
## 第二步修改当前文件目录下的CMakeLists.txt文件内容
```
set(LLVM_TARGET_DEFINITIONS OldDialectTypes.td)

mlir_tablegen(OldDialect.h.inc --gen-dialect-decls)
mlir_tablegen(OldDialect.cpp.inc --gen-dialect-defs)

mlir_tablegen(OldDialectTypes.h.inc --gen-typedef-decls)
mlir_tablegen(OldDialectTypes.cpp.inc --gen-typedef-defs)

add_public_tablegen_target(OldDialectIncGen)
```
## 第三步编写OldDialectTypes.h文件内容
```
#ifndef DIALECT_NORTH_STAR_TYPES_H
#define DIALECT_NORTH_STAR_TYPES_H

#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/Dialect/Tensor/IR/Tensor.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/MLIRContext.h"
//#include "/home/old/Old-project/build/OldDialect/include/Dialect/Old/OldDialect.h.inc"
#define FIX
#define GET_TYPEDEF_CLASSES
#include "/home/old/Old-project/build/OldDialect/include/Dialect/Old/OldDialectTypes.h.inc"
#undef FIX
#endif
```
！！注意开头这个***NORTH_STAR***字样可以修改成你当前命名的Dialect名称，这个宏*GET_TYPEDEF_CLASSES* 的作用可以看[[#补充：对部分宏的作用与使用介绍]]  
## 第四步编写OldDialectTypes.cpp文件内容
```
#include "/home/old/Old-project/OldDialect/include/Dialect/Old/OldDialectTypes.h"
#include "/home/old/Old-project/OldDialect/include/Dialect/Old/OldDialect.h"
#include "/home/old/Old-project/third_party/llvm-project/llvm/include/llvm/ADT/TypeSwitch.h"
#include "/home/old/Old-project/third_party/llvm-project/llvm/include/llvm/Support/LogicalResult.h"
#include "/home/old/Old-project/third_party/llvm-project/llvm/include/llvm/Support/raw_ostream.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/Dialect/Arith/IR/Arith.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/BuiltinTypeInterfaces.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/DialectImplementation.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/OpImplementation.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/Support/LLVM.h"
#define FIX
#define GET_TYPEDEF_CLASSES
#include "/home/old/Old-project/build/OldDialect/include/Dialect/Old/OldDialectTypes.cpp.inc"

namespace mlir::old{
！！！下面是最重要的一步，注册该方言中的类型到MLIR的上下文中。
--------------------------------------------------------------------------------
void OldDialect::registerType(){
    llvm::outs()<<"register: "<<getDialectNamespace()<<"Types \n";
    addTypes<
    #define GET_TYPEDEF_LIST
    #include "/home/old/Old-project/build/OldDialect/include/Dialect/Old/OldDialectTypes.cpp.inc"
    >();
}
---------------------------------------------------------------------------------
::llvm::LogicalResult OldTensorType::verify(::llvm::function_ref<::mlir::InFlightDiagnostic()> emitError, 
                                            ::llvm::ArrayRef<int64_t> shape, Type elementType, int64_t device_id){
    if (device_id < 0) {
        return emitError() << " Invalid device id";
      }
    if (!elementType.isIntOrFloat()) {
        return emitError() << " Invalid element type ";
      }
    return llvm::success();
}
Type OldTensorType::parse(AsmParser &parser){
	**解析类型前缀
    if(parser.parseLess()) return Type(); // 解析 '<'

	**解析形状信息
    SmallVector<int64_t,4> dimensions;
    if(parser.parseDimensionList(dimensions,true,true))  `parseDimensionList` 处理形如 `[1,2]` 或 `1x2` 的维度列表，支持动态维度（用 `?` 表示）
    return Type();

    auto typeLoc = parser.getCurrentLocation();

	**解析元素属性
    Type elementType;
    if(parser.parseType(elementType))   `parseType` 会递归解析嵌套的类型（如 `i32`、`f64` 等）
        return Type();
    if(parser.parseComma())
        return Type();

	**解析额外属性（device_id = 0）
    int device_id = 0;
    if(parser.parseInteger(device_id)){  
        if(parser.parseGreater())
		return Type();  // 解析 '>'
    }

	**创建并返回类型实例
    return parser.getChecked<OldTensorType>(parser.getContext(),dimensions,elementType,device_id);  `getChecked` 会创建 `OldTensorType` 实例，并自动调用 `verify` 方法验证类型合法性
}

void OldTensorType::print(AsmPrinter &printer) const{
    printer << "<";
  for (int64_t dim : getShape()) {
    if (dim < 0) {
      printer << "?" << 'x';
    } else {
      printer << dim << 'x';
    }
  }
  printer.printType(getElementType());
  printer << ",";
  printer << getDeviceId();
  printer << ">";
}
}
#undef FIX
```
其中`parse`是 **类型解析器（Type Parser）** 的实现，用于将文本格式的类型描述转化为内存中的类型对象，负责解析形如 `<shape, element_type, device_id>` 的字符串表示，并构建对应的类型实例（如 `OldTensorType`）。与`print方法`对偶关系，将类型对象转换为文本格式（如 `old_tensor<[1,2], f32, 0>`）

## 第五步编写main.cpp
 ==**其实只需要关注day_2函数部分即可**==
```
#include <cstddef>

#include "/home/old/Old-project/OldDialect/include/Dialect/Old/OldDialect.h"
#include "/home/old/Old-project/OldDialect/include/Dialect/Old/OldDialectTypes.h"

#include <iostream>
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/DialectRegistry.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/MLIRContext.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/Dialect/Arith/IR/Arith.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/BuiltinDialect.h"
#include "/home/old/Old-project/third_party/llvm-project/mlir/include/mlir/IR/BuiltinTypes.h"
void Day_1(){

    mlir::DialectRegistry registr;
    mlir::MLIRContext context(registr);
    auto dialect = context.getOrLoadDialect<mlir::old::OldDialect>();
    dialect->sayHello();

}

void typeBrief() {
    // 文件定义：llvm-project/mlir/include/mlir/IR/BuiltinTypes.td
    auto context = new mlir::MLIRContext;
  
    // 浮点数，每种位宽和标准定义一个
    auto f32 = mlir::Float32Type::get(context);
    llvm::outs() << "F32类型 :\t";
    f32.dump();
  
    auto bf16 = mlir::BFloat16Type::get(context);
    llvm::outs() << "BF16类型 :\t";
    bf16.dump();
  
    // Index 类型，机器相关的整数类型
    auto index = mlir::IndexType::get(context);
    llvm::outs() << "Index 类型 :\t";
    index.dump();
  
    // 整数类型, 参数: 位宽&&有无符号
    auto i32 = mlir::IntegerType::get(context, 32);
    llvm::outs() << "I32 类型 :\t";
    i32.dump();
    auto ui16 = mlir::IntegerType::get(context, 16, mlir::IntegerType::Unsigned);
    llvm::outs() << "UI16 类型 :\t";
    ui16.dump();
  
    // 张量类型,表示的是数据，不会有内存的布局信息。
    auto static_tensor = mlir::RankedTensorType::get({1, 2, 3}, f32);
    llvm::outs() << "静态F32 张量类型 :\t";
    static_tensor.dump();
    // 动态张量
    auto dynamic_tensor =
        mlir::RankedTensorType::get({mlir::ShapedType::kDynamic, 2, 3}, f32);
    llvm::outs() << "动态F32 张量类型 :\t";
    dynamic_tensor.dump();
  
    // Memref类型：表示内存
    auto basic_memref = mlir::MemRefType::get({1, 2, 3}, f32);
    llvm::outs() << "静态F32 内存类型 :\t";
    basic_memref.dump();
    // 带有布局信息的内存
  
    auto stride_layout_memref = mlir::MemRefType::get(
        {1, 2, 3}, f32, mlir::StridedLayoutAttr::get(context, 1, {6, 3, 1}));
    llvm::outs() << "连续附带布局信息的 F32 内存类型 :\t";
    stride_layout_memref.dump();
    // 使用affine 表示布局信息的内存
    auto affine_memref = mlir::MemRefType::get(
        {1, 2, 3}, f32,
        mlir::StridedLayoutAttr::get(context, 1, {6, 3, 1}).getAffineMap());
    llvm::outs() << "连续附带 affine 布局信息的 F32 内存类型 :\t";
    affine_memref.dump();
    // 动态连续附带 affine 布局信息的内存
    auto dynamic_affine_memref =
        mlir::MemRefType::get({mlir::ShapedType::kDynamic, 2, 3}, f32,
                              mlir::StridedLayoutAttr::get(
                                  context, 1, {mlir::ShapedType::kDynamic, 3, 1})
                                  .getAffineMap());
    llvm::outs() << "连续附带 affine 布局信息的动态 F32 内存类型 :\t";
    dynamic_affine_memref.dump();
    // 具有内存层级信息的内存
    auto L1_memref =
        mlir::MemRefType::get({mlir::ShapedType::kDynamic, 2, 3}, f32,
                              mlir::StridedLayoutAttr::get(
                                  context, 1, {mlir::ShapedType::kDynamic, 3, 1})
                                  .getAffineMap(),
                              1);
    llvm::outs() << "处于L1层级的 F32 内存类型 :\t";
    L1_memref.dump();
    /*
    // gpu 私有内存层级的内存
    context->getOrLoadDialect<mlir::gpu::GPUDialect>();
    auto gpu_memref =
        mlir::MemRefType::get({mlir::ShapedType::kDynamic, 2, 3}, f32,
                              mlir::StridedLayoutAttr::get(
                                  context, 1, {mlir::ShapedType::kDynamic, 3, 1})
                                  .getAffineMap(),
                              mlir::gpu::AddressSpaceAttr::get(
                                  context, mlir::gpu::AddressSpace::Private));
    llvm::outs() << "连续附带 affine 布局信息的动态 F32 Gpu Private内存类型 :\t";
    gpu_memref.dump();
  */
    // 向量类型,定长的一段内存
    auto vector_type = mlir::VectorType::get(3, f32);
    llvm::outs() << "F32 1D向量类型 :\t";
    vector_type.dump();
  
    auto vector_2D_type = mlir::VectorType::get({3, 3}, f32);
    llvm::outs() << "F32 2D向量类型 :\t";
    vector_2D_type.dump();
    delete context;
}

void Day_2(){
    std::cout<<"=========================================================================================="<<"\n";
    typeBrief();
    std::cout<<"=========================================================================================="<<"\n";
    // 初始化方言注册器
  mlir::DialectRegistry registry;
  // 初始化上下文环境
  mlir::MLIRContext context(registry);
  // 加载/注册方言
  auto dialect = context.getOrLoadDialect<mlir::old::OldDialect>();
  // 调用方言中的方法
  dialect->sayHello();
  // 静态 NSTensor
  mlir::old::OldTensorType old_tensor = 
      mlir::old::OldTensorType::get(&context, {1, 2, 3},
                                          mlir::Float32Type::get(&context), 3);                               
  llvm::outs() << "Old Tensor 类型 :\t";
  old_tensor.dump();

   // 动态 NSTensor
  mlir::old::OldTensorType dy_old_tensor = 
      mlir::old::OldTensorType::get(&context, {mlir::ShapedType::kDynamic, 2, 3},
                                          mlir::Float32Type::get(&context), 3);
    llvm::outs() << "动态 Old Tensor 类型 :\t";
     dy_old_tensor.dump();

}


int main(){
    //Day_1();
    Day_2();
}
```



## 补充：对部分宏的作用与使用介绍 
![[Pasted image 20250503232144.png]]
-  **`GET_TYPEDEF_CLASSES`** 该宏会导入 TableGen 自动生成的 **类型类定义**（如 `OldTensorType` 的 C++ 类）。
	- 这些生成的类包含：
	    - 类型的构造函数、属性访问器（如 `getShape()`、`getElementType()`）；
	    - 类型的基本接口实现（如 `Type` 基类的虚函数）；
	    - 与 MLIR 框架集成所需的元数据。
-  **`GET_TYPEDEF_LIST`**  **生成类型注册列表**
	- **核心功能：**
		- 该宏会导入 TableGen 自动生成的 **类型名称列表**，用于向 MLIR 上下文注册类型。
		- 在 `registerType()` 函数中，你需要通过 `addTypes<...>()` 注册所有类型，而 `GET_TYPEDEF_LIST` 会展开为这些类型的列表。
	- 例如：
```
void OldDialect::registerType() {
    llvm::outs() << "register: " << getDialectNamespace() << "Types \n";
    addTypes<
    #define GET_TYPEDEF_LIST
    #include "/home/old/Old-project/build/OldDialect/include/Dialect/Old/OldDialectTypes.cpp.inc"
    >();
}
```
**这行代码会展开为类似 `addTypes<OldTensorType, OldVectorType, ...>()` 的形式，将所有定义的类型注册到 MLIR 上下文。**
-  **`GET_OPERATORS`**  用于生成操作（Operations）的类定义。**（暂且不谈，还没有使用过）**

