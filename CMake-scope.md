https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#build-specification-and-usage-requirements

传递用法要求
目标的使用要求可以传递给依赖对象。 target_link_libraries（）命令具有PRIVATE，INTERFACE和PUBLIC关键字来控制传播使用要求。

```cmake
add_library(archive archive.cpp)
target_compile_definitions(archive INTERFACE USING_ARCHIVE_LIB)

add_library(serialization serialization.cpp)
target_compile_definitions(serialization INTERFACE USING_SERIALIZATION_LIB)

add_library(archiveExtras extras.cpp)
target_link_libraries(archiveExtras PUBLIC archive)
target_link_libraries(archiveExtras PRIVATE serialization)
# archiveExtras is compiled with -DUSING_ARCHIVE_LIB
# and -DUSING_SERIALIZATION_LIB

add_executable(consumer consumer.cpp)
# consumer is compiled with -DUSING_ARCHIVE_LIB
target_link_libraries(consumer archiveExtras)
```

因为`archive`库是目标`archiveExtras`的PUBLIC依赖项，所以它（`archive`）的使用要求也传播到了`consumer`。因为`serialization`是`archiveExtras`的`PRIVATE`依赖项，所以它（`serialization`）的使用要求不会传播到`consumer`。
通常，如果只有库的实现而不是头文件使用了依赖项（生成库的cpp文件中使用依赖项，但是库所包含的头文件没有包含依赖项），则应在使用target_link_libraries（）时用PRIVATE关键字指定依赖项。如果在库的头文件中还使用了依赖项（例如，用于类继承），则应将其指定为PUBLIC依赖项。库的实现不使用依赖项，而是仅由其头文件使用的依赖项应指定为INTERFACE依赖项。

可以通过多次使用每个关键字来调用target_link_libraries（）命令：

```cmake
target_link_libraries(archiveExtras
  PUBLIC archive
  PRIVATE serialization
)
```

通过从依赖关系中读取目标属性的INTERFACE变量，并将这些值附加到operad的非INTERFACE变量中来传播使用需求。例如，读取依赖项的INTERFACE_INCLUDE_DIRECTORIES，并将其附加到操作数的INCLUDE_DIRECTORIES上。如果订单是相关的并得以维护，并且由target_link_libraries（）调用产生的订单不允许正确编译，则使用适当的命令直接设置属性可能会更新订单。
例如，如果必须以lib1 lib2 lib3的顺序指定目标的链接库，那么必须以lib3 lib1 lib2的顺序指定include目录：

```cmake
target_link_libraries(myExe lib1 lib2 lib3)
target_include_directories(myExe
PRIVATE $<TARGET_PROPERTY:INTERFACE_INCLUDE_DIRECTORIES:lib3>)
```



 https://cmake.org/cmake/help/latest/command/target_include_directories.html 

target_include_directories
将包含目录添加到目标。

```cmake
target_include_directories(<target> [SYSTEM] [BEFORE]
  <INTERFACE|PUBLIC|PRIVATE> [items1...]
  [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])
```

在编译给定目标时,要指定包含目录。<target>必须由诸如add_executable（）或add_library（）之类的命令创建，并且不能为ALIAS(别名)目标。

如果指定了BEFORE，则内容将被添加到该属性的前面，而不是被附加。

INTERFACE，PUBLIC和PRIVATE关键字来指定以下参数的范围。 PRIVATE和PUBLIC项目将放到<target>的INCLUDE_DIRECTORIES变量里。 PUBLIC和INTERFACE项目将放到<target>的INTERFACE_INCLUDE_DIRECTORIES变量里。 （导入的目标仅支持INTERFACE项。）以下参数指定包含目录。

指定的包含目录可以是绝对路径或相对路径。对相同<target>的重复调用将按调用顺序追加项目。如果指定了SYSTEM，则会在某些平台上告知编译器目录为系统包含目录（对此设置进行信号化可能会产生效果，例如编译器跳过警告，或者在依赖性计算中不考虑这些固定安装的系统文件-请参阅编译器文档）。如果将SYSTEM与PUBLIC或INTERFACE一起使用，则INTERFACE_SYSTEM_INCLUDE_DIRECTORIES目标属性将使用指定的目录填充。

target_include_directories的参数可以使用语法为<< ...>的“生成器表达式”。有关可用表达式，请参见cmake-generator-expressions（7）手册。有关定义构建系统属性的更多信息，请参见cmake-buildsystem（7）手册。

包含目录的使用要求通常在构建树和安装树之间有所不同。 BUILD_INTERFACE和INSTALL_INTERFACE生成器表达式可用于根据使用位置来描述单独的使用要求。相对路径在INSTALL_INTERFACE表达式中允许，并且相对于安装前缀进行解释。例如：

target_include_directories（mylib PUBLIC
 $ <BUILD_INTERFACE：$ {CMAKE_CURRENT_SOURCE_DIR} / include / mylib>
 $ <INSTALL_INTERFACE：include / mylib>＃<前缀> / include / mylib
）
创建可重定位软件包
请注意，建议不要使用指向依赖项包含目录的绝对路径来填充目标的INTERFACE_INCLUDE_DIRECTORIES的INSTALL_INTERFACE。这将把包含依赖项的包含目录路径硬编码到已安装的软件包中，如在制作软件包的计算机上所发现的。

INTERFACE_INCLUDE_DIRECTORIES的INSTALL_INTERFACE仅适合为目标本身提供的标头指定必需的包含目录，而不是为其INTERFACE_LINK_LIBRARIES目标属性中列出的传递性依赖关系提供的目录。这些依赖关系本身应该是在INTERFACE_INCLUDE_DIRECTORIES中指定其自己的标头位置的目标。

请参阅cmake-packages（7）手册的“创建可重定位软件包”部分，以了解在创建用于重新分发的软件包时指定使用要求时必须采取的其他措施。

 https://cmake.org/cmake/help/latest/command/target_link_libraries.html?highlight=target_link_libraries#libraries-for-both-a-target-and-its-dependents 

```cmake
target_link_libraries(<target> <item>...)
```

默认情况下，使用这个函数（signature）的库依赖关系是可传递的（我认为默认是PUBLIC关键词）。 当这个目标（假设为库A，我认为不是库无法被链接，所以如果这是一个单纯的可执行文件，那么这个scope关键词就没有意义）被链接到另一个目标（目标B）时，链接到这个目标（库A）的库也可以被另一个目标（目标B）链接上。

这个可传递的“链接接口”存储在INTERFACE_LINK_LIBRARIES目标属性中，可以直接设置该属性。 当CMP0022未设置为NEW时，将建立传递链接，但可被LINK_INTERFACE_LIBRARIES属性覆盖。 调用此命令的其他签名可能会设置该属性，从而使由该签名专有链接的任何库都变为私有。

