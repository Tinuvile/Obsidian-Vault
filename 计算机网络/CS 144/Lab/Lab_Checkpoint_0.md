# networking warmup

---

## Networking by hand

> 在Web browser中访问[网页](http://cs144.keithw.org/hello)。

接下来用命令手动完成浏览器所做的操作。

```bash
tinuvile@LAPTOP-7PVP3HH3:~$ telnet cs144.keithw.org http
Trying 104.196.238.229...
Connected to cs144.keithw.org.
Escape character is '^]'.
^]
telnet> close
Connection closed.
```

这个命令指示`telnet`程序在我的电脑和`cs144.keithw.org`电脑间建立一个字节流连接，并访问该电脑上运行的特定服务`http`。

然后`GET`会告知服务器URL路径，`Host`输入主机部分，`close`以及句末回车再回车告知服务器已经完成HTTP请求。完整的如下：

```bash
tinuvile@LAPTOP-7PVP3HH3:~$ telnet cs144.keithw.org http
Trying 104.196.238.229...
Connected to cs144.keithw.org.
Escape character is '^]'.
GET /hello HTTP/1.1
Host: cs144.keithw.org
Connection: close

HTTP/1.1 200 OK
Date: Tue, 22 Apr 2025 10:04:29 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain

Hello, CS144!
Connection closed by foreign host.
```

---

## Listening and connecting

这个是作为一个简单服务器的角色。在一个终端窗口中输入：

```bash
tinuvile@LAPTOP-7PVP3HH3:~$ netcat -v -l -p 9090
Listening on 0.0.0.0 9090
Connection received on localhost 54172
```

然后在另一个终端中输入：

```bash
tinuvile@LAPTOP-7PVP3HH3:~$ telnet localhost 9090
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
```

这时候第一个终端窗口会出现：

```bash
Connection received on localhost 59892
```

表明已经连接成功了。`netcat`是服务器端，`telnet`是客户端。

![](C:\Users\ASUS\AppData\Roaming\marktext\images\2025-04-22-19-45-39-image.png)

然后在`netcat`窗口`ctrl+C`退出程序，`telnet`也会立即退出。

---

## Writing a network program using an OS stream socket

这个实验要用到**Linux**内核提供的`stream socket`功能，这个功能可以在两台计算机间创建可靠的双向字节流。

实验设计基于Unix套接字API，所以需要在Linux或者WSL环境下完成。我用的是`Ubuntu 20.04.6 LTS`。

```bash
tinuvile@LAPTOP-7PVP3HH3:~/CS144Lab$ cmake -S . -B build
-- The CXX compiler identification is GNU 13.1.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Setting build type to 'Debug'
-- Building in 'Debug' mode.
-- Configuring done (1.7s)
-- Generating done (0.1s)
-- Build files have been written to: /home/tinuvile/CS144Lab/build
tinuvile@LAPTOP-7PVP3HH3:~/CS144Lab$ cmake --build build
[ 26%] Building CXX object util/CMakeFiles/util_debug.dir/address.cc.o
[ 26%] Building CXX object src/CMakeFiles/minnow_debug.dir/byte_stream_helpers.cc.o
[ 26%] Building CXX object util/CMakeFiles/util_debug.dir/debug.cc.o
[ 26%] Building CXX object util/CMakeFiles/util_debug.dir/eventloop.cc.o
[ 26%] Building CXX object src/CMakeFiles/minnow_debug.dir/byte_stream.cc.o
[ 31%] Building CXX object apps/CMakeFiles/stream_copy.dir/bidirectional_stream_copy.cc.o
[ 36%] Building CXX object util/CMakeFiles/util_debug.dir/file_descriptor.cc.o
[ 42%] Building CXX object util/CMakeFiles/util_debug.dir/random.cc.o
[ 47%] Building CXX object tests/CMakeFiles/minnow_testing_debug.dir/common.cc.o
[ 52%] Building CXX object util/CMakeFiles/util_debug.dir/helpers.cc.o
[ 57%] Building CXX object util/CMakeFiles/util_debug.dir/socket.cc.o
[ 63%] Linking CXX static library libminnow_debug.a
[ 63%] Built target minnow_debug
[ 68%] Linking CXX static library libstream_copy.a
[ 68%] Built target stream_copy
[ 73%] Linking CXX static library libminnow_testing_debug.a
[ 78%] Linking CXX static library libutil_debug.a
[ 78%] Built target minnow_testing_debug
[ 78%] Built target util_debug
[ 84%] Building CXX object apps/CMakeFiles/webget.dir/webget.cc.o
[ 89%] Building CXX object apps/CMakeFiles/tcp_native.dir/tcp_native.cc.o
[ 94%] Linking CXX executable tcp_native
[100%] Linking CXX executable webget
[100%] Built target tcp_native
[100%] Built target webget
```

编译源代码。

> 课程给我们提供了一些编码建议，希望充分利用新特性实现最大程度的安全编程，可查阅[C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)。核心理念是确保每个对象的设计具备尽可能小的公开接口，包含大量内部安全检查机制，难以被误用，并能自主清理资源。具体来说：
> 
> - 避免成对操作，如`malloc/free`,`new/delete`，因为这些操作的后者可能因为函数提前返回或抛出异常而无法执行。建议采用**Resource acquisition is initialization**的模式（`RAII`），操作应在对象构造函数中完成，逆向操作则应在析构函数中自动执行。
> 
> - 尽量不要使用原始指针，必要时使用智能指针`unique_ptr`或`shared_ptr`。
> 
> - 避免使用模板、线程、锁和虚函数。
> 
> - 避免使用C风格字符串`char *str`或字符串函数，改用`std::string`。
> 
> - 切勿使用C风格类型转换，必要时使用C++的`static_cast`。
> 
> - 函数参数传递优先使用常量引用方式，如`const Address & address`。
> 
> - 所有不需要修改的变量和方法都声明为`const`。
> 
> - 避免使用全局变量，尽可能为每个变量赋予最小的作用域。
> 
> - 提交作业前运行`cmake --build build --target tidy`获取改进建议，并运行`cmake --build build --target format`保持代码格式一致。

### Reading the Minnow support code

公共接口部分阅读`util/socket.hh`和`util/file_descriptor.hh`。继承关系为`TCPSocket` → `Socket` → `FileDescriptor`，关键部分列在下面：

#### file_descriptor.hh

这个类是一个引用计数的文件描述符句柄，用`shared_ptr`管理`FDWrapper`，`FDWrapper`封装了底层的文件描述符。然后这个类将 POSIX 系统调用封装成了 C++ 方法。

```c
class FileDescriptor {
public:
  // 核心操作
  void read( std::string& buffer );               // 读取数据到字符串
  size_t write( std::string_view buffer );        // 写入字符串数据
  void close();                                   // 显式关闭描述符

  // 状态查询
  int fd_num() const;                             // 获取底层文件描述符编号
  bool eof() const;                               // 检查是否到达文件末尾
  bool closed() const;                            // 检查是否已关闭

  // 高级功能
  void set_blocking( bool blocking );             // 设置阻塞/非阻塞模式
  FileDescriptor duplicate() const;               // 复制描述符（共享底层 FDWrapper）
};
```

#### socket.hh

`Socket`继承自`FileDescriptor`，增加了网络相关的方法，子类`TCPSocket`、`UDPSocket`进一步特化。

```c
class Socket : public FileDescriptor {
public:
  // 基础操作
  void bind( const Address& address );            // 绑定本地地址
  void connect( const Address& address );         // 连接到远程地址
  void shutdown( int how );                       // 关闭连接方向（SHUT_RD/SHUT_WR）

  // 地址信息
  Address local_address() const;                  // 获取本地绑定地址
  Address peer_address() const;                   // 获取对端地址

  // 高级选项
  void set_reuseaddr();                           // 允许地址重用（SO_REUSEADDR）
};

class TCPSocket : public Socket {
public:
  void listen( int backlog = 16 );                // 开始监听连接
  TCPSocket accept();                             // 接受新连接
};

class UDPSocket : public DatagramSocket {
public:
  void sendto( const Address& dest, std::string_view payload ); // 发送数据报
  void recv( Address& src, std::string& payload );              // 接收数据报
};
```

### Writing webget

```c
void get_URL( const string& host, const string& path )
{
  // cerr << "Function called: get_URL(" << host << ", " << path << ")\n";
  // cerr << "Warning: get_URL() has not been implemented yet.\n";
  // 创建 TCP 套接字并连接目标地址
  TCPSocket socket;
  socket.connect( Address( host, "http" ) );

  // 构建 HTTP 请求头
  const string request = "GET " + path + " HTTP/1.0\r\n"
                       + "Host: " + host + "\r\n"
                       + "Connection: close\r\n\r\n";

  // 发送 HTTP 请求
  socket.write( request );

  // 读取直到连接关闭
  string buffer;
  while ( !socket.eof() ) {
    socket.read(buffer);
    cout << buffer;
  }

  // 关闭套接字
  socket.close();
}
```

HTTP 请求是通过 TCP 连接发送的纯文本字节流，`write`方法将字节流写入文件描述符。

运行测试：

```c
tinuvile@LAPTOP-7PVP3HH3:~/CS144Lab$ cd build
tinuvile@LAPTOP-7PVP3HH3:~/CS144Lab/build$  ./apps/webget cs144.keithw.org /hello
HTTP/1.1 200 OK
Date: Fri, 09 May 2025 06:27:15 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain


Hello, CS144!
```

运行`cmake --build build --target check_webget`通过。

```bash
tinuvile@LAPTOP-7PVP3HH3:~/CS144Lab$ cmake --build build --target check_webget
Test project /home/tinuvile/CS144Lab/build
    Start 1: compile with bug-checkers
1/2 Test #1: compile with bug-checkers ........   Passed    0.09 sec
    Start 2: t_webget
2/2 Test #2: t_webget .........................   Passed    1.14 sec

100% tests passed, 0 tests failed out of 2

Total Test time (real) =   1.24 sec
Built target check_webget
```

---

## An in-memory reliable byte stream

这个任务需要我们实现字节流功能。这个字节流是有限容量的，写入的数据不能超过当前可用容量。当写入者写入数据时，如果缓冲区已满，只能写入部分数据。读取者可以读取数据，并从缓冲区中移除已读取的部分，这样释放空间，让写入者可以继续写入。同时，写入者可以关闭流，读取者在读取完所有数据后会到达EOF。

```c
class Writer : public ByteStream
{
public:
  void push( std::string data ); // Push data to stream, but only as much as available capacity allows.
  void close();                  // Signal that the stream has reached its ending. Nothing more will be written.

  bool is_closed() const;              // Has the stream been closed?
  uint64_t available_capacity() const; // How many bytes can be pushed to the stream right now?
  uint64_t bytes_pushed() const;       // Total number of bytes cumulatively pushed to the stream
};

class Reader : public ByteStream
{
public:
  std::string_view peek() const; // Peek at the next bytes in the buffer
  void pop( uint64_t len );      // Remove `len` bytes from the buffer

  bool is_finished() const;        // Is the stream finished (closed and fully popped)?
  uint64_t bytes_buffered() const; // Number of bytes currently buffered (pushed and not popped)
  uint64_t bytes_popped() const;   // Total number of bytes cumulatively popped from stream
};
```

首先`ByteStream`需要维护一个缓冲区，我选用的是`std::string`。

```c
class ByteStream
{
public:
  explicit ByteStream( uint64_t capacity );

  // Helper functions (provided) to access the ByteStream's Reader and Writer interfaces
  Reader& reader();
  const Reader& reader() const;
  Writer& writer();
  const Writer& writer() const;

  void set_error() { error_ = true; };       // Signal that the stream suffered an error.
  bool has_error() const { return error_; }; // Has the stream had an error?

protected:
  // Please add any additional state to the ByteStream here, and not to the Writer and Reader interfaces.
  std::string buffer_;      // Buffer to store data
  uint64_t start_position_; // Position of the start of the buffer
  uint64_t bytes_pushed_;   // Number of bytes pushed to the buffer
  uint64_t bytes_popped_;   // Number of bytes popped from the buffer
  bool closed_;             // Flag indicating whether the stream has been closed
  bool eof_;                // Flag indicating whether the stream has reached its ending
  uint64_t capacity_;       // Maximum capacity of the buffer
  bool error_ {};
};
```

然后实现`Reader`和`Writer`的具体函数，比较简单。

```c
#include "byte_stream.hh"

using namespace std;

ByteStream::ByteStream( uint64_t capacity )
  : buffer_()
  , start_position_( 0 )
  , bytes_pushed_( 0 )
  , bytes_popped_( 0 )
  , closed_( false )
  , eof_( false )
  , capacity_( capacity )
  , error_( false )
{}

void Writer::push( string data )
{
  if ( closed_ || error_ )
    return;

  const uint64_t available = available_capacity();
  const uint64_t byte_to_push = min( data.size(), available );

  if ( byte_to_push > 0 ) {
    buffer_.append( data.substr( 0, byte_to_push ) );
    bytes_pushed_ += byte_to_push;
  }
}

void Writer::close()
{
  closed_ = true;
}

bool Writer::is_closed() const
{
  return closed_;
}

uint64_t Writer::available_capacity() const
{
  return capacity_ - ( buffer_.size() - start_position_ );
}

uint64_t Writer::bytes_pushed() const
{
  return bytes_pushed_;
}

string_view Reader::peek() const
{
  return string_view( buffer_ ).substr( start_position_ );
}

void Reader::pop( uint64_t len )
{
  len = min( len, buffer_.size() - start_position_ );
  start_position_ += len;
  bytes_popped_ += len;

  // if data_read is greater than 4 KB and half of the buffer is used,
  // remove the unused part of the buffer
  if ( start_position_ > 4096 && start_position_ >= buffer_.size() / 2 ) {
    buffer_ = buffer_.substr( start_position_ );
    start_position_ = 0;
  }
}

bool Reader::is_finished() const
{
  return closed_ && ( start_position_ == buffer_.size() );
}

uint64_t Reader::bytes_buffered() const
{
  return buffer_.size() - start_position_;
}

uint64_t Reader::bytes_popped() const
{
  return bytes_popped_;
}
```

运行测试。

```bash
tinuvile@LAPTOP-7PVP3HH3:~/CS144Lab$ cmake --build build --target check0
Test project /home/tinuvile/CS144Lab/build
      Start  1: compile with bug-checkers
 1/11 Test  #1: compile with bug-checkers ........   Passed    6.11 sec
      Start  2: t_webget
 2/11 Test  #2: t_webget .........................   Passed    1.16 sec
      Start  3: byte_stream_basics
 3/11 Test  #3: byte_stream_basics ...............   Passed    0.01 sec
      Start  4: byte_stream_capacity
 4/11 Test  #4: byte_stream_capacity .............   Passed    0.01 sec
      Start  5: byte_stream_one_write
 5/11 Test  #5: byte_stream_one_write ............   Passed    0.01 sec
      Start  6: byte_stream_two_writes
 6/11 Test  #6: byte_stream_two_writes ...........   Passed    0.01 sec
      Start  7: byte_stream_many_writes
 7/11 Test  #7: byte_stream_many_writes ..........   Passed    0.09 sec
      Start  8: byte_stream_stress_test
 8/11 Test  #8: byte_stream_stress_test ..........   Passed    0.02 sec
      Start 37: no_skip
 9/11 Test #37: no_skip ..........................   Passed    0.01 sec
      Start 38: compile with optimization
10/11 Test #38: compile with optimization ........   Passed    2.61 sec
      Start 39: byte_stream_speed_test
        ByteStream throughput (pop length 4096): 20.07 Gbit/s
        ByteStream throughput (pop length 128):  14.28 Gbit/s
        ByteStream throughput (pop length 32):   11.13 Gbit/s
11/11 Test #39: byte_stream_speed_test ...........   Passed    0.14 sec

100% tests passed, 0 tests failed out of 11

Total Test time (real) =  10.16 sec
Built target check0
```

提交前用`clang-format`统一代码风格。

```c
tinuvile@LAPTOP-7PVP3HH3:~/CS144Lab$ cmake --build build --target format
[100%] Formatting source code...
[100%] Built target format
```

这样就算完成了。
