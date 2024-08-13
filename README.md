ROS_PROTOBUF
#总览
技术栈：c++，c++模板，shell，docker，protobuf，cmake，ros序列化库，特征库
项目简介:
ros-protobuf-bridge是一个基于ROS和Protobuf的桥接项目，旨在实现 ROS 消息和 protobuf 消息之间的兼容和互操作性。
主要特点和贡献:
1. 构建环境自动化: 使用 docker 构建整个项目环境，通过 dockerfile 安装 ROS-Noetic 组件、protobuf、cmake 等依赖项，同时使用 Shell 脚本编写第三方库源码安装和容器操作脚本，以实现项目构建流程的自动化和部署的便利性。
2. 兼容性和可扩展性: 基于C++模板编程中的 SFINAE 机制，修改了 roscpp 的核心库 roscpp_serialization 和 roscpp_traits 的底层代码。这样，ros-protobuf-bridge可以同时兼容ROS原生msg和Protobuf消息。
3. 项目管理和构建: 使用 cmake 作为项目的构建系统，编写 proto 测试文件生成相应的静态库，供 ros 自定义功能模块调用。简化了项目的管理和构建过程，并提供灵活的扩展机制，便于后续添加复杂数据类型。
4. 插件化设计: ros-protobuf-bridge 可以作为一个插件嵌入到各种复杂的ROS功能项目中。通过将该项目中的cmake 指令集成到目标项目中，可以轻松地实现基于proto数据的发布和订阅。



## 1.通过项目中dockerfile文件，构建项目镜像 

```bash
cd ~/work/ros_protobuf_msg/docker/build
docker build --network host -t ros_protobuf:noetic  -f ros_x86.dockerfile .
```

## 2.进入docker容器

```bash
cd ~/work/ros_protobuf_msg/docker/scripts
#启动容器
./ros_docker_run.sh
#进入容器
./ros_docker_into.sh
```

## 3.编译代码

```bash
#创建build目录
mkdir build
cd build
cmake ..
make -j6
```

## 4.启动程序

```bash
#先启动roscore，并且启动pb_talker节点
cd /work
source devel/setup.bash
roscore &
rosrun myproject pb_talker
```

```bash
#打开新终端，再次进入容器，启动pb_listener节点
#进入容器中
cd ~/work/ros_protobuf_msg/docker/scripts
#进入容器
./ros_docker_into.sh
rosrun myproject pb_listener
```

#proto文件最终生成了什么
当运行 protocol buffer compiler 编译test.proto时，编译器会以您选择的语言生成代码，您需要使用文件中描述的消息类型，包括获取和设置字段值、将消息序列化为输出流，并从输入流解析消息。
- 对于C++，编译器会根据每个 生成一个.h和.cc文件 .proto，其中包含文件中描述的每种消息类型的类。
- 对于Java，编译器会生成一个.java文件，其中包含每个消息类型的类，以及Builder用于创建消息类实例的特殊类。
protobuf compiler
```bash
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR  path/to/file.proto
--cpp_out生成 C++ 代码存储在DST_DIR
--java_out生成 Java 代码存储在DST_DIR
```



protobuf_generate
cmake 指令，把proto文件生成pb.h、pb.cc
- APPEND_PATH— 一个标志，导致所有原型模式文件的基本路径被添加到IMPORT_DIRS.
- LANGUAGE— 单个值：cpp 或 python。确定正在生成什么类型的源文件。
- OUT_VAR— CMake 变量的名称，该变量将填充生成的源文件的路径。
- EXPORT_MACRO— 应用于所有生成的 Protobuf 消息类和extern变量的宏的名称。
- PROTOC_OUT_DIR—生成的源文件的输出目录，默认为CMAKE_CURRENT_BINARY_DIR.
- PLUGIN— 可选的插件可执行文件。例如，这可能是通往 的路径grpc_cpp_plugin。
- PLUGIN_OPTIONS— 为插件提供的附加选项，例如generate_mock_code=truegRPC cpp 插件。
- TARGET— 生成的文件将作为源添加到提供的目标。
- PROTOS— 原型模式文件列表。如果省略，则将使用每个以protoof结尾的源文件。TARGET
- IMPORT_DIRS— 模式文件的公共父目录。例如，如果架构文件是proto/helloworld/helloworld.proto且导入目录是proto/，则生成的文件是${PROTOC_OUT_DIR}/helloworld/helloworld.pb.cc.
- GENERATE_EXTENSIONS— 如果LANGUAGE省略，则必须将其设置为protoc生成的扩展。
- PROTOC_OPTIONS— 转发到protoc调用的其他参数。



- ##定义proto文件
test.proto
```protobuf
syntax = "proto3";
package monitor.proto;

message CpuLoad {
    float load_avg_1 = 1;
    float load_avg_3 = 2;
    float load_avg_15 = 3;
  }
  
message NetInfo {
    string name = 1;
    float send_rate = 2;
  }
  
 message MonitorInfo{
  CpuLoad cpu_load = 1; #std::string
  repeated NetInfo net_info = 2; # std::vector
}
```

proto数据类型
![output (2)](https://github.com/user-attachments/assets/3507e991-dc52-418f-926d-3cb33f08805f)


protoc编译器，编译test.proto
```shell
protoc -I$SRC_DIR --cpp_out=$DST_DIR test.proto
#private-node/proto
参数解释
-I$SRC_DIR 指定test.proto所在目录
--cpp_out=$DST_DIR 指定生成cpp相关文件，并指定生成路径
test.proto 需要编译的proto文件
```



ProtoBuf 的 C++ API 来读写消息
三大类序列化，反序列化API
```Plain Text
bool SerializeToString(std::string* output) const; //将消息序列化并储存在指定的string中。注意里面的内容是二进制的，而不是文本；我们只是使用string作为一个很方便的容器。
“1001001000111” std::string 序列化消息存储为string

bool ParseFromString(const std::string& data); //从给定的string解析消息。


bool SerializeToArray(void * data, int size) const        //将消息序列化至数组
uint8 * data 

bool ParseFromArray(const void * data, int size)        //从数组解析消息

bool SerializeToOstream(ostream* output) const; //将消息写入到给定的C++ ostream中。

bool ParseFromIstream(istream* input); //从给定的C++ istream解析消息。
```

```C++
syntax = "proto3";
package monitor.proto;

message CpuLoad {
    float load_avg_1 = 1;
    float load_avg_3 = 2;
    float load_avg_15 = 3;
  }
  
message NetInfo {
    string name = 1;
    float send_rate = 2;
  }
  
 message MonitorInfo{
  std::string happly = 1;
  CpuLoad cpu_load = 2; #std::string
  repeated NetInfo net_info = 3; # std::vector
}


monitor::proto::MonitorInfo monitor_info;
monitor_info.set_happly("1111");

::monitor::proto::CpuLoad* cpu_load_msg = monitor_info.mutable_cpu_load();
cpu_load_msg->set_load_avg_1(1.2);
cpu_load_msg->set_load_avg_3(1.4);
cpu_load_msg->set_load_avg_15(1.8);

::monitor::proto::NetInfo*  net_info_msg1  = monitor_info.add_net_info();
net_info_msg1->set_name("shiki-1");
net_info_msg1->set_send_rate(12.5);

auto net_info_msg2  = monitor_info.add_net_info();
net_info_msg2->set_name("shiki-2");
net_info_msg2->set_send_rate(8.5);

 // 对消息对象MonitorInfo序列化到string容器
std::string serializedStr;
monitor_info.SerializeToString(&serializedStr);
std::cout<<"serialization result:"<<serializedStr<<std::endl; //序列化后的字符串内容是二进制内容，非可打印字符，预计输出乱码


//反序列化
monitor::proto::MonitorInfo monitor_info;
monitor_info.ParseFromString(serializedStr)；
    
std::cout << monitor_info.happly()<<std::endl;


message CpuLoad {
    float load_avg_1 = 1;
    float load_avg_3 = 2;
    float load_avg_15 = 3;
  }
  
auto cpu_load_parse =  monitor_info.cpu_load();
std::cout << cpu_load_parse.load_avg_1()<< cpu_load_parse.load_avg_3()<<std::endl;


for (int i = 0; i < monitor_info.net_info_size(); i++) {
        std::cout <<monitor_info.net_info(i).name();
         std::cout << monitor_info.net_info(i).send_rate();
}
```



#性能相关：
Benchmark
以下数据来自https://code.google.com/p/thrift-protobuf-compare/wiki/Benchmarking
解析性能
![output](https://github.com/user-attachments/assets/0bacfa51-0abe-4009-b4e0-9bec7e07a5e8)
序列化之空间开销
![output (1)](https://github.com/user-attachments/assets/39d598ab-c2ae-4ec9-9644-f7f05878ef81)
从上图可得出如下结论：
1、XML序列化（Xstream）无论在性能和简洁性上比较差。
2、Thrift与Protobuf相比在时空开销方面都有一定的劣势。
3、Protobuf和Avro在两方面表现都非常优越。
选型建议
以上描述的五种序列化和反序列化协议都各自具有相应的特点，适用于不同的场景：

1、对于公司间的系统调用，如果性能要求在100ms以上的服务，基于XML的SOAP协议是一个值得考虑的方案。

2、基于Web browser的Ajax，以及Mobile app与服务端之间的通讯，JSON协议是首选。对于性能要求不太高，或者以动态类型语言为主，或者传输数据载荷很小的的运用场景，JSON也是非常不错的选择。
3、对于调试环境比较恶劣的场景，采用JSON或XML能够极大的提高调试效率，降低系统开发成本。

4、当对性能和简洁性有极高要求的场景，Protobuf，Thrift，Avro之间具有一定的竞争关系。

5、对于T级别的数据的持久化应用场景，Protobuf和Avro是首要选择。如果持久化后的数据存储在Hadoop子项目里，Avro会是更好的选择。

6、由于Avro的设计理念偏向于动态类型语言，对于动态语言为主的应用场景，Avro是更好的选择。

7、对于持久层非Hadoop项目，以静态类型语言为主的应用场景，Protobuf会更符合静态类型语言工程师的开发习惯。

8、如果需要提供一个完整的RPC解决方案，Thrift是一个好的选择。

9、如果序列化之后需要支持不同的传输层协议，或者需要跨防火墙访问的高性能场景，Protobuf可以优先考虑。

