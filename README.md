## 一个跨平台的C++串口库

### 简介

**本库是一个跨平台简单的C++串口工具库, 其目的是简化串口开发**

- 简单易用, 只需注册回调即可实现异步接收数据.
- 跨平台, 支持*Windows / Linux / macOS*等主流操作系统.

库文件在[serial_port](serial_port)文件夹下

### 使用教程

1. 拷贝[`serial_port`](serial_port)到项目文件夹中.

2. 在CMakeLists.txt中添加以下内容:

   ```cmake
   add_subdirectory(serial_port)    # 添加子文件夹
   
   add_executable(serialportDemo main.cpp)                   # 根据实际项目更改
   target_link_libraries(serialportDemo PRIVATE serialport)  # 链接serialport
   ```

3. 在源文件中包含头文件`#include "serialport/serialport.h"`即可使用.

### 使用案例

该案例在[main2.cpp](main2.cpp)中

```cpp
#include <fstream>
#include <iostream>

#include "serialport/serialport.h"

int main()
{
  // 列出所有可用的串口, 这个接口不是很全面, 仅供参考
  std::vector<serial::PortInfo> ports = SerialPort::listPorts();
  if (ports.empty())
  {
    std::cout << "No serial ports found." << std::endl;
    return 0;
  }
  std::cout << "Available serial ports:" << std::endl;
  for (const auto& port : ports)
  {
    std::cout << "Port: " << port.port << " | Description: " << port.description
              << " | Hardware ID: " << port.hardware_id << std::endl;
  }

  //   // 根据操作系统选择串口名称
  //   std::string port_name;
  // #if defined(_WIN32) || defined(_WIN64)
  //   port_name = "COM5";  // Windows 示例
  // #else
  //   port_name = "/dev/ttyS000";  // Linux 示例
  // #endif

  // 让用户输入串口号
  std::string port_name;
  std::cout << "Enter the serial port name to use (e.g., COM5 or /dev/ttyUSB0): ";
  std::getline(std::cin, port_name);

  // 打开文件用于记录数据
  std::ofstream log_file("serial_log.txt", std::ios::out | std::ios::app);
  if (!log_file.is_open())
  {
    std::cerr << "Failed to open log file!" << std::endl;
    return 1;
  }

  // 创建串口对象
  SerialPort sp;

  // 配置串口参数并注册回调 (回调函数在独立线程中调用)
  sp.setPort(port_name)
    .setBaudRate(115200)
    .setTimeout(10)                                                           // 读超时 10ms
    .setReconnectLimit(3)                                                     // 最大重连 3 次
    .setLogCallback([](SerialPort::LogLevel level, const std::string& msg) {  // 日志回调
      std::string levelStr;
      switch (level)
      {
      case SerialPort::LogLevel::Info:
        levelStr = "[INFO] ";
        break;
      case SerialPort::LogLevel::Warning:
        levelStr = "[WARN] ";
        break;
      case SerialPort::LogLevel::Error:
        levelStr = "[ERROR] ";
        break;
      }
      std::cout << levelStr << msg << std::endl;
    })
    .setDataCallback([&log_file](const std::string& data) {
      for (unsigned char c : data)
      {
        log_file << std::hex << std::uppercase << (int)c << " ";
      }
      log_file.flush();                   // 保证立即写入文件
      log_file << std::dec << std::endl;  // 每条数据换行，并恢复十进制格式
    });

  // 打开串口
  if (!sp.open())
  {
    std::cerr << "Failed to open serial port" << std::endl;
    return 1;
  }

  std::cout << "Serial port opened. Type messages to send. Type 'exit' to quit.\n";

  // 循环读取用户输入并发送
  std::string line;
  while (true)
  {
    std::cout << "> ";
    std::getline(std::cin, line);

    if (line == "exit") break;

    sp.write(line + "\r\n");  // 发送回车换行，可按需要调整
  }

  // 关闭串口
  sp.close();

  return 0;
}
```


控制台运行结果:
```sh
PS ***\SerialPort\build_output\bin> .\serialportTest.exe
Available serial ports:
Port: COM1 | Description: Electronic Team Virtual Serial Port (COM1->COM2) | Hardware ID: EVSERIAL9
Port: COM2 | Description: Electronic Team Virtual Serial Port (COM2->COM1) | Hardware ID: EVSERIAL9
Enter the serial port name to use (e.g., COM5 or /dev/ttyUSB0): COM1
[INFO] [COM1@115200] SerialPort opened
Serial port opened. Type messages to send. Type 'exit' to quit.
> hello world
> hello serialport
> exit
[INFO] [COM1@115200] SerialPort closed
PS ***\SerialPort\build_output\bin> 
```

串口工具截图:



输出文件`serial_log.txt`内容:

```txt
57 65 6C 63 6F 6D 65 20 74 6F 20 55 61 72 74 41 73 73 69 73 74 
AA BB CC DD EE FF 
```

### 说明

本库**使用了这个三方库: [serial](https://github.com/wjwwood/serial.git)**

- 该库跨平台,足够轻量且稳定, 已在大量知名项目中使用(如ROS)

- 只包含改库的`include`和`src`文件夹
- 简化了原来自带的CMakeLists.txt
- 也可以直接使用原生serial库, 案例在[main.cpp](main.cpp)中