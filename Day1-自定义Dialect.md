# 编写CMake文件
==还有一些前置信息我放在其他文档中了==[[Day2-自定义DialectType]]<-中的正文部分
在项目的根目录中编写整体的CMake文件，内容如下
```
# 设置 CMake 最低版本要求

cmake_minimum_required(VERSION 3.15)

  

# 定义项目名称和支持的语言

project(OldDialect LANGUAGES C CXX)

  

# 设置 C 和 C++ 标准

set(CMAKE_C_STANDARD 17)

set(CMAKE_CXX_STANDARD 20)


# 设置构建类型为 Release

set(CMAKE_BUILD_TYPE "Release" CACHE STRING " " FORCE)

  

# 添加编译选项

add_compile_options(-fPIC)

add_compile_options(-fno-rtti)

  

# 假设 llvm-project 的构建目录路径

set(LLVM_BUILD_DIR "/home/richard/data/Old-project/third_party/llvm-project/build")

  

# 查找 LLVM 和 MLIR

find_package(LLVM REQUIRED CONFIG HINTS ${LLVM_BUILD_DIR}/lib/cmake/llvm)

find_package(MLIR REQUIRED CONFIG HINTS
${LLVM_BUILD_DIR}/lib/cmake/mlir)

  

# 输出 LLVM 和 MLIR 的信息

message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")

message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

message(STATUS "Found MLIR ${MLIR_PACKAGE_VERSION}")

message(STATUS "Using MLIRConfig.cmake in: ${MLIR_DIR}")

  

# 添加 LLVM 和 MLIR 的包含目录

include_directories(${LLVM_INCLUDE_DIRS})

include_directories(${MLIR_INCLUDE_DIRS})

  

# 添加 LLVM 和 MLIR 的定义

add_definitions(${LLVM_DEFINITIONS})

  

# 添加 MLIR 的 CMake 模块目录到搜索路径

set(MLIR_CMAKE_MODULE_DIR "${LLVM_BUILD_DIR}/lib/cmake/mlir")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${MLIR_CMAKE_MODULE_DIR})

  

# 包含 LLVM 和 MLIR 的 CMake 模块

include(TableGen)

include(AddLLVM)

include(AddMLIR)

  

# 添加子目录

add_subdirectory(OldDialect)

add_executable(Old-1 "test.cpp")

target_link_libraries(Old-1 MLIROldDialect)
```
大致流程是在/build目录中使用`cmake -G"Ninja" ..` 以及`ninja Old-1`将两个文件链接起来

在该目录中对`data/Old-project/OldDialect/include/Dialect/Old/CMakeLists.txt`进行编写，在build目录中使用`ninja OldDialectIncGen`生成相应的inc文件

```
set(LLVM_TARGET_DEFINITIONS OldDialect.td)

  

mlir_tablegen(OldDialect.h.inc --gen-dialect-decls)

mlir_tablegen(OldDialect.cpp.inc --gen-dialect-defs)

  

add_public_tablegen_target(OldDialectIncGen)
```

最后对`data/Old-project/OldDialect/src/Dialect/Old/CMakeLists.txt`路径下的文件进行编辑
```
add_mlir_dialect_library(MLIROldDialect

  

OldDialect.cpp

  

ADDITIONAL_HEADER_DIRS

    ${CMAKE_CURRENT_SOURCE_DIR}/../include/Dialect/Old

  

    DEPENDS

    OldDialectIncGen

  

    LINK_LIBS PUBLIC

    MLIRIR

    MLIRTensorDialect
)

```
# TabeGen编写
```

#ifndef DIALECT_NORTH_STAR
#define DIALECT_NORTH_STAR

使用该路径头文件
include "/home/richard/data/Old-project/third_party/llvm-project/mlir/include/mlir/IR/DialectBase.td"
使用该路径文件为模板/home/richard/data/Old-project/third_party/llvm-project/mlir/include/mlir/IR/DialectBase.td

def OldDialect:Dialect{

  // The name of the dialect.

  let name = "Old";
  

  // Short summary of the dialect.

  let summary = "My Dialect";

  

  // The description of the dialect.

  let description = "My dialect";

  

  //方言的依赖

  let dependentDialects = ["::mlir::tensor::TensorDialect"];

  

  //命名使用空间

  let cppNamespace = "::mlir::old";  


  //额外的声明

  let extraClassDeclaration = [{

    static void sayHello();

  }];
```

# 编写头文件和实现文件

```
OldDialect.h

#ifndef DIALECT_NORTH_STAR_H
#define DIALECT_NORTH_STAR_H

#include "mlir/Dialect/Tensor/IR/Tensor.h"

#include "mlir/IR/MLIRContext.h"

#include "/home/richard/data/Old-project/build/OldDialect/include/Dialect/Old/OldDialect.h.inc"

  

#endif
```

```
OldDialect.cpp

#include "Dialect/Old/OldDialect.h"

#define FIX

#include "Dialect/Old/OldDialect.cpp.inc"

namespace mlir::old{

//初始化方法完成

    void OldDialect::initialize(){

        llvm::outs()<<"initialize"<<getDialectNamespace()<<"\n";

    }

  

    OldDialect::~OldDialect(){

        llvm::outs()<<"destroying"<<getDialectNamespace()<<"\n";

    }

  

    void OldDialect::sayHello(){

        llvm::outs()<<"Hello "<<getDialectNamespace()<<"\n";

    }

}

  

#undef FIX
```
额外声明会在inc.h文件中生成，在.cpp文件中将其实现
![[Pasted image 20250428102328.png]]

# 主函数文件编辑
```
#include "/home/richard/data/Old-project/OldDialect/include/Dialect/Old/OldDialect.h"

#include "/home/richard/data/Old-project/third_party/llvm-project/mlir/include/mlir/IR/DialectRegistry.h"

  

void test(){

  

    mlir::DialectRegistry registry;

    mlir::MLIRContext context(registry);

    auto dialect = context.getOrLoadDialect<mlir::old::OldDialect>();

    dialect->sayHello();

}

  

int main(){

    test();

    return 0;

}
```

