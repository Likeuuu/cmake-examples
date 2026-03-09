## Cmake 理念

不往全局加东西, 而是**精准**往某个**target**加属性



# GitHub 项目 cmake-examples

## project

- 设置项目名，这里系统给了一些其他**默认的变量**⭐⭐⭐
  - 工程相关
    -  PROJECT_NAME
    -  PROJECT_SOURCE_DIR
    -  PROJECT_BINARY_DIR
  - 语言相关
    -  CMAKE_C_STANDARD
    -  CMAKE_CXX_STANDARD
    -  CMAKE_CXX_COMPILER_ID
    -  CMAKE_CXX_COMPILER_VERSION
  - 平台相关
    -  CMAKE_SYSTEM_NAME        # Linux / Windows / Darwin
    -  CMAKE_SYSTEM_PROCESSOR   # x86_64 / arm / aarch64
- 注意:
  - CMake文件或者其内容改动时（包括文件路径、编译选项等）， 需要**全编译** --- ???

## add

如何将项目代码**节点化**

- add_subdirectory
- add_library
- add_executable



## target

如何解决**全局污染**问题

解决add**节点化**后,之间如何**依赖**的问题

- target_include_directories, 让编译器找到头文件**真实路径**
  - PUBLIC / PRIVATE / INTERFACE⭐⭐
- target_sources
- target_link_libraries



## 路径变量

| Variable                     | Info                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| CMAKE_SOURCE_DIR             | The root source directory                                    |
| CMAKE_CURRENT_SOURCE_DIR⭐⭐⭐⭐ | The current source directory if using sub-projects and directories. |
| PROJECT_SOURCE_DIR           | The source directory of the current cmake project.           |
| CMAKE_BINARY_DIR             | The root binary / build directory. This is the directory where you ran the cmake command. |
| CMAKE_CURRENT_BINARY_DIR     | The build directory you are currently in.                    |
| PROJECT_BINARY_DIR           | The build directory for the current project.                 |



- CMAKE_SOURCE_DIR
  - 项目顶层目录(**宿主项目**顶层CMAKE的目录, 三方库顶层cmake**禁止**使用)
  - 注意:
    - **应该用**?
      - 在**宿主项目**顶层用
    - 子模块**不应该用**这种找路径
      - 子模块的lib如果被用到其他项目,会找不到文件. **破坏**解耦原则
      - 子模块**不应该知道**
        - 谁是顶层
        - 顶层工程名字
        - 顶层的目录结构
- CMAKE_CURRENT_SOURCE_DIR⭐⭐
  - 当前Cmake的位置
  - 解决问题:
    - 子模块能**完美解耦**路径依赖
- PROJECT_SOURCE_DIR: 最近的project路径
  - 为什么设计?
    - 描述**项目**的**根**路径
    - 在**顶层**使用
  - 1个工程里,可能1级project套着两个2级project.
    - Cmake**如何知道**PROJECT_SOURCE_DIR是哪个project路径?
      - 当前cmake离得最近的project的PROJECT_SOURCE_DIR
- CMAKE_BINARY_DIR, CMAKE_CURRENT_BINARY_DIR, PROJECT_BINARY_DIR
  - build的路径,用于存放中间文件




## 添加cpp

- aux_source_directory

  - ~~~cmake
    aux_source_directory(src SRCS)   # 不递归
    add_library(mylib ${SRCS})
    ~~~

  - **好处**: 很多cpp场景顺手

  - **坏处**:

    - **不支持增量**编译,  有cpp改动需要删除build再cmake
    - 违背清晰原则, 工业级**不推荐使用**

- 改进方式:

  - targe_sources

    - ~~~cmake
      add_library(mylib)
      target_sources(mylib
            PRIVATE
              src/a.cpp
                src/b.cpp
        )
        # 如果把属性从 PRIVATE 改为 PUBLIC 没有意义, 因为源文件不会传播
      ~~~

  - file(GLOB ...) **底线**

    - ~~~cmake
      file(GLOB SRCS CONFIGURE_DEPENDS src/*.cpp)
      add_library(mylib ${SRCS})
      
      # CONFIGURE_DEPENDS 能感知文件变化而重新编译
      
      # 补充 递归扫描
      file(GLOB_RECURSE SRCS CONFIGURE_DEPENDS src/*.cpp)  
      ~~~

## 其它

- **message**打印

  - 基本: 

    - ```
      message("变量名: ${变量名}")
      ```

  - 常用:

    - ```
      message(STATUS "状态信息: ${变量名}")
      message(WARNING "警告信息: ${变量名}")
      message(AUTHOR_WARNING "作者警告: ${变量名}")
      message(SEND_ERROR "错误信息: ${变量名}")
      message(FATAL_ERROR "致命错误: ${变量名} - 会停止执行")
      ```

      

## 问题

- add_library的其他用法

  - 大项目喜欢写
    - add_library(core OBJECT)

- ~~~cmake
  # 顶层 CMakeLists.txt
  add_library(mylib)
  add_subdirectory(src)
  add_subdirectory(third_party)
  
  # src/CMakeLists.txt
  target_sources(mylib PRIVATE a.cpp b.cpp)
  
  # third_party/CMakeLists.txt
  target_sources(mylib PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/extra.cpp)
  ~~~

  - 非顶层的两个地方都对mylib 进行赋属性. 最后是不是mylib 具备了所有属性? 
    - **是的**，最后 `mylib` 会**累积**(变量**唯一**性)所有子目录中添加的属性
  - 如果add_library(mylib)不在third_party的父结构, 而在同级的子结构, 是否编译会出错
    - **不一定**, 看被链接前, 是否被创建
      - **target**  具有**全局唯一性**!!!, 所以不依赖只能看子结构!!!, 全局都能访问
      - 但在依赖前, 一定要**先被创建** (因为是cmake是**顺序执行**的)
    - 万一现在项目很大又发生这种情况,**怎么办??**

- 现在有add_library(liblxm); 然后在项目100个地方都在target_sources. **问题是**, 在项目的某个地方去链接这个liblxm时, 这时编译器能知道其它还未target_sources 的liblxm吗 ? (属性如何具备完整性)

  - 可以.
  - CMake 分两大阶段：
    - configure（配置阶段）
      - CMake 会 **完整执行**所有 CMakeLists.txt
      - 构建一个 **完整的** target graph ( 解决了提问)
    - build（构建阶段）
      - 整理完所有配置后, 获取完整的信息, **开始构建**

- 其他地方如何正确引用头**文件??**

- 三个关**键字??**

  - 这三个关键字在 CMake 中是统一语义：

    | 命令                       | PRIVATE    | PUBLIC      | INTERFACE   |
    | -------------------------- | ---------- | ----------- | ----------- |
    | target_sources             | 只编译本库 | 几乎没意义  | header-only |
    | target_include_directories | 只自己用   | 自己+依赖者 | 只给依赖者  |
    | target_compile_definitions | 只自己     | 自己+依赖者 | 只依赖者    |
    | target_link_libraries      | 只自己     | 自己+依赖者 | 只依赖者    |
