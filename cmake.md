## Cmake 目的

- 解决**跨平台编译**问题
  - 统一描述项目结构
  - 然后生成不同平台的构建系统

- **构建图**(通过声明 **Target** 和**依赖关系**)而非**构建命令**(cmake ..配置 + 生成)
- 根据平台开始构建


## Cmake 理念

不往全局加东西, 而是**精准**往某个**target**加属性

target**两种**属性

- 构建产物(为自己)
- Usage Requirements(为别人)


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

## add  (创建target)

如何将项目代码**节点化**

- add_subdirectory
- add_library
- add_executable





## target(类型)

- **STATIC**
  - **优点**:
    - 运行时不依赖库, 部署简单, 不存在版本冲突. 代码保密
  - **缺点**:
    - 文件大,  多个程序内存**无法共享**
  - 使用**场景**
    - 核心**基础**模块(不怎么变化)
    - 部署麻烦的场景
- **SHARED**
  - **优点**:
    - 文件小, 多个程序可以**共享内存**, 库热更新
  - **缺点**:
    - 部署麻烦
  - 使用**场景**
    - **插件**(容易变化)
    - SDK/API
    - 系统级库
- INTERFACE
  - 目的: 申明依赖关系,并不参与构建
- IMPORTED
  - 外部库
- OBJECT
  - **优点**:
    - 减少编译次数
    - 减少.a文件
  - **缺点**:
    - 不能被链接 : $<TARGET_OBJECTS:core>
- ALIAS





## target (配置target属性)

如何解决**全局污染**问题

解决add**节点化**后,之间如何**依赖**的问题

- target_include_directories, 让编译器找到头文件**真实路径**
  - PUBLIC / PRIVATE / INTERFACE⭐⭐
- target_sources
- target_link_libraries
- target_compile_options



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

---

## 三个关键字

### INTERFACE

**目的**: 为别人服务(target**第二**属性), 不参与生成(target**第一**属性). 

​	 声明依赖传播规则

**场景**:

- **工具类**函数 (只有头文件)
- 利用申明依赖, 去组成**库组**(解决重复写代码问题)  -- 指定链接规则. 但自身不产生构建



---


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

- add其它

  - add_dependencies
  - add_custom_command / add_custom_target

- target 其它

  - target_compile_options
    - 例子: target_compile_options(mylib PUBLIC -Wall)
  - target_compile_features
    - 例子: target_compile_features(mylib PUBLIC cxx_std_20)

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

- 现在有add_library(liblxm); 然后在项目100个地方都在target_sources. **问题是**, 在项目的某个地方去链接这个liblxm时, 这时编译器能知道其它还未target_sources 的liblxm吗 ? (属性如何具备完整性)

  - 可以.
  - CMake 分两大阶段：
    - configure（配置阶段）
      - CMake 会 **完整执行**所有 CMakeLists.txt
      - 构建一个 **完整的** target graph ( 解决了提问)
    - build（构建阶段）
      - 整理完所有配置后, 获取完整的信息, **开始构建**

- 三个关**键字??**

  - 这三个关键字在 CMake 中是统一语义：

    | 命令                       | PRIVATE    | PUBLIC      | INTERFACE   |
    | -------------------------- | ---------- | ----------- | ----------- |
    | target_sources             | 只编译本库 | 几乎没意义  | 只给依赖者  |
    | target_include_directories | 只自己用   | 自己+依赖者 | header-only |
    | target_compile_definitions | 只自己     | 自己+依赖者 | 只依赖者    |
    | target_link_libraries      | 只自己     | 自己+依赖者 | 只依赖者    |

- 例子

  - 如何引用头文件? 头文件冲突问题 ? 什么是 header-only ? 

  - ~~~tex
    project/
    ├── app/
    │   └── app.c
    ├── math/
    │   ├── include/
    │   │   └── math.h
    │   └── src/
    │       └── math.c
    └── utils/
        └── include/
            └── log.h
            
    ~~~

  - ~~~cmake
    # utils 库
    add_library(utils INTERFACE)
    target_include_directories(utils INTERFACE 
        ${CMAKE_CURRENT_SOURCE_DIR}/utils/include
    )
    
    # math 库
    add_library(math math/src/math.c)
    target_include_directories(math PUBLIC 
        ${CMAKE_CURRENT_SOURCE_DIR}/math/include
    )
    target_link_libraries(math PUBLIC utils)  # 传递依赖
    
    # app 可执行文件
    add_executable(app app/app.c)
    target_link_libraries(app PRIVATE math)  # 自动获得所有必要的包含路径
    ~~~

  - 为什么app **能引用**util下的**头文件**?

    - 得益于Cmake设计的传递依赖: math 知道 utils 的信息. 所以 app 链接了 math, 就知道utils了

  - 如果app链接的东西中, 有**多个log.h**冲突怎么办?

    - 需要依赖**好的项目结构**

      - 目录命名空间: (大公司解决方案)

        - ~~~tex
          原来:
          utils/
          └── include/
              └── log.h
          
          
          改为
          utils/
          └── include/
              └── utils/
                  └── log.h
          ~~~

  - header-only:

    - 什么是header-only
      - 库的属性是 **INTERFACE**. 只给别人用
    - 什么场景用?
      - 申明和实现**都在.h 文件**
      - **工具类**使用



## 注意:

- 对于引用其它地方文件, 最好**不要直接**写路径. 而是通过**库的形式**去链接

~~~cmake
core/
  utils.cpp
  math.cpp

libA/
  algos.cpp

libB/
  filters.cpp
~~~

- 例题: .o, .a 关系? 数量关系?

  - ~~~cpp
    add_library(core STATIC
        utils.cpp
        math.cpp
    )
        
    // 生成 .a
    // 里面其实是:
    // core.a
    //  ├── utils.o
    //  └── math.o
        
    add_library(libA STATIC
        algos.cpp
    )
    
    target_link_libraries(libA PUBLIC core)
    //生成：
    libA.a
    
    //里面只有：
    libA.a
     └── algos.o
    
    //注意：
    
    // 没有 utils.o
    // 没有 math.o
    // 虽然有 target_link_libraries(libA core)
    // 但core 的代码没有被复制进去。
        
    // 为什么?
    // 静态库不会链接, 只会记录依赖关系. 真正的链接发生在  
        // 最终可执行文件
        
        
        
    // 最终程序出现时
    add_executable(app main.cpp)
    
    target_link_libraries(app
        libA
        libB
    )
    
    // 链接器会这样处理：
    
    main.o
    algos.o
    filters.o
    utils.o
    math.o
    
    // 它会从：
    
    libA.a
    libB.a
    core.a
    
    //中 挑选需要的 .o。
    
    // 最终：
    
    app
     ├── main.o
     ├── algos.o
     ├── filters.o
     ├── utils.o
     └── math.o
    ~~~

  - .a 是一堆.o的**组合**

  - 工业级构建规则 :

    - 库 = 只包含自己的 .o
    - 依赖库：只在**最终链接**时合并
    - 链接器规则 : **同一个符号**只能存在**一份**

  - 以上是争对**STAITC场景**

  - **SHARED场景**

    - ~~~c++
      add_library(core SHARED utils.cpp math.cpp)
      // 生成：
      
      // Linux
      
      // libcore.so
      
      // 程序：
      
      // app -> core.so
      
      // 最终：
      
      // app (不包含 core)
      // core.so (单独文件)
      
      // 运行时：
      
      app
       └── load core.so
      
      // 特点：
      
      // 多个程序共享一份 core.so
      
      // 例如：
      
      app1
      app2
      app3
      
      // 全部：
      
      // 使用同一个 core.so
      
      // 这就是动态库的最大特点：
      
      // 运行时共享代码
          
      ~~~

    - 只能是一份,没有多份的情况

    - **多个**程序可以连**同一块** 动态库内存

  - **OBJECT 场景**:

    - ~~~cmake
      add_library(core OBJECT
          utils.cpp
          math.cpp
      )
      # 生成：
      
      utils.o
      math.o
      
      # 但是：
      
      # 不会生成 core.a
      
      # 当你这样用：
      
      add_library(libA
          algos.cpp
          $<TARGET_OBJECTS:core>
      )
      
      add_library(libB
          filters.cpp
          $<TARGET_OBJECTS:core>
      )
      
      # 最终：
      
      libA
       ├── algos.o
       ├── utils.o
       └── math.o
      
      libB
       ├── filters.o
       ├── utils.o
       └── math.o
      
      #注意：
      
      #utils.o 被复制到了两个库
      
      #所以：
      
      OBJECT library
      #不是共享
      #而是复用编译结果
      
      #特点：
      
      # 编译一次
      # 复制使用
      
      ~~~

    - 编译一次, 复制使用

  - INTERFACE

    - 完全没有代码, 指定传播规则
    - 与**.a不同**, 是一堆**未管理**的 **.o 集合**
