# Protocol Buffers（Protobuf）入门学习指南

> 面向初学者的核心梳理：是什么 → 为什么 → 原理 → 怎么用 → 速查手册。
> 目标：读完本文后，你能看懂 `.proto` 文件、理解二进制编码原理，并独立完成一次"定义 → 生成代码 → 序列化/反序列化"的完整流程。

---

## 目录

1. [什么是 Protobuf](#1-什么是-protobuf)
2. [为什么需要它（动机）](#2-为什么需要它动机)
3. [核心原理](#3-核心原理)
   - 3.1 [整体工作流程](#31-整体工作流程)
   - 3.2 [.proto 文件：接口契约（IDL）](#32-proto-文件接口契约idl)
   - 3.3 [二进制编码原理（Wire Format）](#33-二进制编码原理wire-format)
   - 3.4 [反序列化与向后兼容](#34-反序列化与向后兼容)
   - 3.5 [代码生成](#35-代码生成)
4. [快速上手：从零开始](#4-快速上手从零开始)
5. [proto3 语法核心速查](#5-proto3-语法核心速查)
6. [与 JSON / XML 对比](#6-与-json--xml-对比)
7. [优点与缺点](#7-优点与缺点)
8. [版本演进：proto2 → proto3 → Editions](#8-版本演进proto2--proto3--editions)
9. [常见疑问（FAQ）](#9-常见疑问faq)
10. [学习路线与官方资源](#10-学习路线与官方资源)

---

## 1. 什么是 Protobuf

**Protocol Buffers（简称 Protobuf）** 是 Google 开发的一种**与语言无关、与平台无关、可扩展的序列化数据格式**，用于在程序之间高效地存储和交换结构化数据。

它诞生于 2001 年（Google 内部），2008 年开源，现在是微服务、移动端、存储系统等领域的事实标准之一。

一个比喻帮助理解：

> Protobuf 就像一套"**标准化的快递系统**"：
> - `.proto` 文件 = **双方签订的合同**，约定货物（数据）的结构；
> - 二进制编码 = **高效打包方式**，把货物按规则压成小体积的包裹；
> - `protoc` 编译器 = **自动装卸工具**，根据合同为每种语言生成打包/拆包的代码。

Protobuf 实际包含三个部分：

| 组成部分 | 作用 |
| --- | --- |
| **接口描述语言（IDL）** | 用 `.proto` 文件描述数据结构，类似于"数据库建表语句" |
| **二进制序列化格式** | 把数据编码成紧凑的二进制字节流（wire format） |
| **代码生成器（protoc）** | 根据 `.proto` 自动生成各语言的读写代码 |

它和 JSON、XML 属于同一类东西——都是"把内存中的结构化数据变成一串字节（序列化），再把这串字节还原成数据（反序列化）"的格式，只是 Protobuf 走的是**紧凑二进制**路线。

---

## 2. 为什么需要它（动机）

假设你写一个服务，要把下面这条数据发给另一个服务：

```json
{"id": 150, "name": "Lily", "email": "lily@example.com"}
```

用 JSON 文本传输，会遇到几个问题：

1. **体积大**：字段名 `"id"`、`"name"` 每次都重复传输，浪费带宽；
2. **解析慢**：接收方要逐字符扫描文本、处理引号和转义；
3. **没有契约**：字段叫什么、什么类型，全靠口头约定，容易出错；
4. **类型弱**：数字可能被当作文本处理，跨语言时容易出现精度问题。

Protobuf 的解决思路：

| 问题 | Protobuf 的解法 |
| --- | --- |
| 体积大 | 用**字段编号**（1、2、3…）代替字段名，用二进制代替文本 |
| 解析慢 | 二进制格式按字节直接读取，无需文本扫描 |
| 没有契约 | `.proto` 文件就是强类型的**共享契约**，编译期即可校验 |
| 类型弱 | 强类型定义，跨语言类型映射明确 |

这也解释了它的三个核心卖点：**更小、更快、更强类型**。

---

## 3. 核心原理

### 3.1 整体工作流程

```
① 编写契约          ② 编译生成代码          ③ 在代码中使用
┌────────────┐      ┌──────────────┐      ┌─────────────────────┐
│ person.proto│ ───> │   protoc     │ ───> │ person_pb2.py  (Python)│
│ 定义消息结构 │      │  编译器       │      │ person.pb.go   (Go)    │
│            │      │  生成多语言代码 │      │ Person.java    (Java) │
└────────────┘      └──────────────┘      └─────────────────────┘
                                                   │
                            序列化（对象→字节）      │      反序列化（字节→对象）
                        msg.SerializeToString()   │   msg.ParseFromString()
                        proto.Marshal(m)          ▼   proto.Unmarshal(b, m)
                                            ┌───────────┐
                                            │ 二进制字节流 │
                                            │ 08 96 01…  │
                                            └───────────┘
```

**关键点**：`.proto` 文件是唯一的"事实来源"（Single Source of Truth）。发送方和接收方可以是用不同语言编写的程序，只要用同一份 `.proto` 生成代码，就能互相通信。

### 3.2 .proto 文件：接口契约（IDL）

一个最小的 `.proto` 文件长这样：

```proto
syntax = "proto3";   // 声明使用 proto3 语法

message Person {
  int32  id    = 1;  // 字段类型 + 字段名 + 字段编号
  string name  = 2;
  string email = 3;
}
```

- `message`：定义一条消息（相当于一个类 / 一张表 / 一个 struct）；
- 每个字段由三部分组成：**类型 + 名称 + 唯一编号**；
- **字段编号（field number）**是 Protobuf 的灵魂：编码时二进制流里只存编号，不存字段名，所以编号一旦分配就**永远不要复用**。

### 3.3 二进制编码原理（Wire Format）

这是 Protobuf 最核心的原理。二进制流把每条消息编码为一系列 **字段记录**，每条记录结构如下：

```
┌──────────┬──────────┐
│   tag    │  value   │
│ (键：编号+类型) │ (值：具体数据) │
└──────────┴──────────┘
```

#### ① Tag：字段编号 + 线类型

```
tag = (field_number << 3) | wire_type
```

tag 本身也用 varint 编码。其中 `wire_type`（线类型）只有 6 种：

| 线类型 | 值 | 含义 | 用于哪些字段 |
| --- | --- | --- | --- |
| Varint | 0 | 变长整数 | int32/int64/uint32/uint64/sint32/sint64/bool/enum |
| 64-bit | 1 | 固定 8 字节 | fixed64/sfixed64/double |
| Length-delimited | 2 | 长度前缀 + 数据 | string/bytes/嵌套消息/packed 数组 |
| Start group | 3 | 已废弃 | （不再使用） |
| End group | 4 | 已废弃 | （不再使用） |
| 32-bit | 5 | 固定 4 字节 | fixed32/sfixed32/float |

> 例：字段编号 1 的 int32 → tag = (1<<3)|0 = 8 = `0x08`；
> 字段编号 2 的 string → tag = (2<<3)|2 = 18 = `0x12`。

#### ② Varint：变长整数编码

整数按 **7 位一组**拆分，从低位到高位，每组的**最高位（MSB）作为"是否还有下一字节"的标记**：

- MSB = 1：后面还有字节；
- MSB = 0：这是最后一个字节。

**手工演算：编码 150**

```
150 的二进制 = 1001 0110
按 7 位分组： 001 0110 | 000 0001
             （低 7 位）  （高 1 位）
字节 1：1 001 0110 → 0x96（MSB=1，还有后续）
字节 2：0 000 0001 → 0x01（MSB=0，结束）
结果：0x96 0x01（2 字节）
```

**优点**：小数字用 1 个字节（<128），大数字才需要更多字节。int32 最多 5 字节，int64 最多 10 字节。绝大多数实际数据都是小数字，所以非常省空间。

#### ③ ZigZag：让负数也变小

普通 int32 存负数时，负数会符号扩展成 64 位，**任何负数都要 10 个字节**。ZigZag 把负数"折叠"成小正数：

```
sint32: (n << 1) ^ (n >> 31)
sint64: (n << 1) ^ (n >> 63)

 0 →  0     1 →  2
-1 →  1    -2 →  3
 2 →  4    -3 →  5   （绝对值小的数映射结果也小）
```

所以 `sint32`/`sint64` 适合**可能有负数**的字段（如温度、余额变化量）。

#### ④ Length-delimited：字符串、嵌套消息、数组

格式为：`tag + 长度(varint) + 原始内容`。

- `string`：长度 + UTF-8 字节；
- 嵌套 `message`：长度 + 该消息自己的编码字节（递归结构）；
- `repeated` 标量字段（proto3 默认 packed）：把多个值连续编码后整体打包，只写一个 tag 和一个总长度。

#### ⑤ 完整实例：手工编码一条消息

消息定义：

```proto
message Person {
  int32  id   = 1;
  string name = 2;
}
```

待编码数据：`id = 150`，`name = "Lily"`。

**第 1 步：编码 id 字段**

```
tag   = (1<<3)|0 = 0x08          （字段1 + Varint 类型）
value = varint(150) = 0x96 0x01  （上面已算过）
→ 08 96 01
```

**第 2 步：编码 name 字段**

```
tag    = (2<<3)|2 = 0x12         （字段2 + Length-delimited）
length = 4 → 04
内容    = "Lily" 的 UTF-8 字节 = 4C 69 6C 79
→ 12 04 4C 69 6C 79
```

**最终二进制流（共 9 字节）**：

```
08 96 01 12 04 4C 69 6C 79
```

对比 JSON 文本 `{"id":150,"name":"Lily"}` 共 **22 字节**——Protobuf 节省了约 60%，字段越多、名字越长，优势越大。

#### ⑥ 为什么它又小又快（总结）

| 设计 | 带来的效果 |
| --- | --- |
| 用字段编号代替字段名 | 省掉大量重复文本 |
| varint 变长整数 | 小整数只占 1 字节 |
| 长度前缀 | 无需扫描，直接跳过/定位任意字段 |
| 默认值不编码 | 未设置的字段完全不占字节 |
| 字段之间无分隔符、可任意顺序 | 解析器按 tag 直接"跳读"，无回溯 |
| 二进制直读 | 无需像 JSON 那样解析文本、处理转义 |

### 3.4 反序列化与向后兼容

反序列化时，解析器**逐个读取 tag**：

1. 从 tag 中解出字段编号和线类型；
2. 按线类型读取 value；
3. 在自己的 schema 里找到对应编号的字段，赋值；
4. **遇到不认识的字段编号 → 直接跳过**（因为长度可从线类型推知）；
5. **缺少的字段 → 用默认值填充**。

第 4、5 条规则是 Protobuf 向后兼容的基石：

- **新代码读旧数据**：新字段缺失 → 用默认值，没问题；
- **旧代码读新数据**：不认识的新字段 → 跳过，其他字段照常解析。

因此演化规则只有一条铁律：**字段编号永不复用**。可以加字段、可以删字段（用 `reserved` 防止编号被再次使用），但不要"换个含义复用旧编号"。

```proto
message Person {
  reserved 4, 15, 9 to 11;      // 永久保留这些编号
  reserved "old_field_name";    // 也保留旧字段名
}
```

### 3.5 代码生成

`protoc` 读取 `.proto` 文件，先把它编译成一个通用的描述对象（`FileDescriptorProto`），再交给**各语言的插件**生成代码：

```
person.proto ──protoc──> FileDescriptorProto ──> [python 插件] ──> person_pb2.py
                                               ──> [go 插件]    ──> person.pb.go
                                               ──> [java 插件]  ──> Person.java
                                               ──> [grpc 插件]  ──> 服务端/客户端骨架
```

生成的代码包含：消息类、字段的 getter/setter、`Serialize`/`Parse` 方法等。你写业务代码时，操作的就是这些**强类型对象**，而不是手动拼字节。

---

## 4. 快速上手：从零开始

### 4.1 安装 protoc 编译器

**方式一：官方下载**（推荐，Windows 也适用）
从 <https://github.com/protocolbuffers/protobuf/releases> 下载对应平台的压缩包，解压后把 `bin` 目录加入 `PATH`，然后验证：

```bash
protoc --version   # 例如：libprotoc 36.x
```

**方式二：包管理器**

```bash
# macOS
brew install protobuf
# Ubuntu/Debian
sudo apt install -y protobuf-compiler
```

**方式三：用 Python 全家桶（新手最省事）**
`grpcio-tools` 自带 protoc 编译器：

```bash
pip install protobuf grpcio-tools
python -m grpc_tools.protoc --version
```

### 4.2 编写第一个 .proto 文件

创建 `person.proto`：

```proto
syntax = "proto3";

package demo;              // 包名，防止不同 .proto 之间的符号冲突

message Person {
  int32  id     = 1;
  string name   = 2;
  string email  = 3;
  repeated string phones = 4;   // 数组：0 个或多个

  message Address {             // 嵌套消息
    string city   = 1;
    string street = 2;
  }
  Address address = 5;
}
```

### 4.3 生成代码

```bash
# Python（用 grpcio-tools，无需单独装 protoc）
python -m grpc_tools.protoc -I . --python_out=. person.proto
# 生成 person_pb2.py

# 也可以用官方 protoc
protoc -I . --python_out=. person.proto

# Go（需先安装插件）
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
protoc -I . --go_out=. --go_opt=paths=source_relative person.proto
# 生成 person.pb.go

# 其他语言同理：--cpp_out、--java_out、--csharp_out、--ruby_out、--php_out …
```

### 4.4 Python 使用示例（完整可运行）

```python
# demo.py
import person_pb2

# ---- 构造对象并序列化（对象 → 字节）----
p = person_pb2.Person()
p.id = 150
p.name = "Lily"
p.email = "lily@example.com"
p.phones.append("138-0000-0000")
p.phones.append("010-12345678")
p.address.city = "Beijing"

data = p.SerializeToString()
print(f"序列化后：{data}")          # b'\x08\x96\x01\x12\x04Lily...'
print(f"字节数：{len(data)} 字节")

# ---- 反序列化（字节 → 对象）----
p2 = person_pb2.Person()
p2.ParseFromString(data)
print(p2.name)                     # Lily
print(p2.phones)                   # ['138-0000-0000', '010-12345678']
print(p2.address.city)             # Beijing

# ---- 调试技巧 ----
print(p)                           # 人类可读的文本形式
```

命令行调试小工具：

```bash
# 把二进制流"破译"出来看（不需要 .proto 文件）
protoc --decode_raw < person.bin
# 输出：1: 150  2: "Lily"  4: "138-0000-0000" …

# 有 .proto 时，按字段名展示
protoc --decode=demo.Person person.proto < person.bin
```

### 4.5 Go 使用示例（对照）

```go
package main

import (
    "fmt"
    "google.golang.org/protobuf/proto"
)

func main() {
    p := &personpb.Person{
        Id:    150,
        Name:  "Lily",
        Email: "lily@example.com",
        Address: &personpb.Person_Address{City: "Beijing"},
    }

    data, _ := proto.Marshal(p)      // 序列化
    fmt.Printf("%x (%d bytes)\n", data, len(data))

    var p2 personpb.Person
    _ = proto.Unmarshal(data, &p2)   // 反序列化
    fmt.Println(p2.GetName(), p2.GetAddress().GetCity())
}
```

### 4.6 配合 gRPC 定义服务接口

`.proto` 不仅能描述数据，还能描述**远程调用接口**（这就是 gRPC）：

```proto
syntax = "proto3";
package search;

message SearchRequest {
  string query = 1;
  int32  page  = 2;
}

message SearchResponse {
  repeated string results = 1;
}

// 定义一个 RPC 服务
service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

配合 `--grpc_out` 插件即可生成服务端/客户端骨架，**Protobuf 负责数据格式，gRPC 负责网络传输**，二者常被一起使用。

---

## 5. proto3 语法核心速查

### 5.1 标量类型一览

| proto 类型 | 说明 | 编码方式 | Go | Python | Java |
| --- | --- | --- | --- | --- | --- |
| `double` | 双精度浮点 | 64-bit 定长 | float64 | float | double |
| `float` | 单精度浮点 | 32-bit 定长 | float32 | float | float |
| `int32` | 有符号整数，负数编码低效 | varint | int32 | int | int |
| `int64` | 有符号整数，负数编码低效 | varint | int64 | int | long |
| `uint32` / `uint64` | 无符号整数 | varint | uint32/uint64 | int | int/long |
| `sint32` / `sint64` | ZigZag 编码，**负数友好** | varint | int32/int64 | int | int/long |
| `fixed32` / `fixed64` | 固定 4/8 字节，**大数字友好** | 定长 | uint32/uint64 | int | int/long |
| `sfixed32` / `sfixed64` | 定长有符号 | 定长 | int32/int64 | int | int/long |
| `bool` | 布尔 | varint（0/1） | bool | bool | boolean |
| `string` | UTF-8 字符串 | length-delimited | string | str | String |
| `bytes` | 任意字节序列 | length-delimited | []byte | bytes | ByteString |

选型口诀：**可能有负数用 sint，大且固定的数用 fixed，否则 int32/int64 就够**。

### 5.2 字段规则

```proto
message Example {
  string name   = 1;        // 普通字段（singular）：0 或 1 个
  optional string nickname = 2;  // 可选字段：可区分"未设置"与"空字符串"（proto3.12+）
  repeated int32 scores = 3;     // 数组：0 个或多个（proto3 默认 packed 编码）

  map<string, int32> dict = 4;   // 映射表（键不能用枚举/浮点/bytes）

  oneof choice {                 // 多选一：同一时刻只有一个字段有值
    string text  = 5;
    int32  number = 6;
  }
}
```

### 5.3 枚举、导入与保留

```proto
enum Status {
  STATUS_UNSPECIFIED = 0;   // proto3 规定：枚举第一个值必须是 0
  STATUS_ACTIVE     = 1;
  STATUS_INACTIVE   = 2;
}

import "google/protobuf/timestamp.proto";   // 导入其他 .proto
import "common.proto";

message User {
  string name = 1;
  Status status = 2;
  google.protobuf.Timestamp created_at = 3;  // 官方内置的"时间戳"类型
  common.Address addr = 4;                    // 使用导入的消息
}
```

### 5.4 proto3 默认值规则

| 类型 | 未设置时的默认值 |
| --- | --- |
| 数值 | `0` |
| string | `""`（空串） |
| bool | `false` |
| bytes | 空字节 |
| enum | 第一个枚举值（必须是 0） |
| message | 空对象（多数语言中表现为 null/None 语义） |

> **注意**：proto3 中默认值字段**不会被编码进字节流**。所以"字段没赋值"和"字段恰好等于默认值"在字节流上无法区分——需要区分时用 `optional` 关键字。

### 5.5 字段编号规则

- 合法范围：**1 ~ 536,870,911**；
- **19000 ~ 19999** 是 Protobuf 内部保留区间，不能用；
- 1 ~ 15 的编号编码后 tag 只占 1 字节，**给最常用字段**；16 以上 tag 需要 2 字节。

---

## 6. 与 JSON / XML 对比

| 维度 | Protobuf | JSON | XML |
| --- | --- | --- | --- |
| 可读性 | ✗ 二进制，人不可读 | ✓ 文本可读 | ✓ 文本可读 |
| 体积 | ★★★ 最小（约 JSON 的 1/3 ~ 1/10） | 中等 | 最大 |
| 序列化速度 | ★★★ 最快 | 中等 | 最慢 |
| 反序列化速度 | ★★★ 最快 | 较快 | 慢 |
| Schema 契约 | ✓ 强类型，编译期检查 | ✗ 无（靠约定/JSON Schema 补充） | 可用 DTD/XSD |
| 类型安全 | ✓ 强 | ✗ 弱（数字/字符串易混） | 一般 |
| 向后兼容 | ✓ 设计内建（字段编号机制） | 靠约定 | 靠约定 |
| 自描述性 | ✗ 需要 .proto 才能看懂 | ✓ 字段名自带 | ✓ 标签自带 |
| 浏览器直接使用 | ✗ 需额外工具 | ✓ 原生支持 | 一般 |
| 调试成本 | 高（需工具解码） | 低 | 低 |

**一句话选型**：服务间通信、高性能场景、需要强契约 → Protobuf（常配 gRPC）；对外 API、浏览器交互、需要人来读日志 → JSON。

---

## 7. 优点与缺点

**优点**

1. **体积小**：字段编号 + varint + 省略默认值，压缩率高；
2. **速度快**：二进制直读，编码/解码开销远低于文本格式；
3. **强类型契约**：`.proto` 文件即文档即校验，编译期发现错误；
4. **跨语言**：官方支持 C++/Java/Python/Go/C#/Ruby/PHP/Objective-C/Kotlin/Dart/Rust 等，一份契约到处用；
5. **向后兼容**：字段编号机制让新旧版本程序可以互相通信；
6. **生态成熟**：gRPC 默认格式，云原生领域广泛使用。

**缺点**

1. **不可读**：二进制数据无法直接用眼睛检查，调试需要工具；
2. **需要编译步骤**：修改 `.proto` 后要重新生成代码，比 JSON"即写即用"繁琐；
3. **自描述性差**：没有 `.proto` 文件，字节流几乎无法解读；
4. **浏览器支持差**：前端不能像 JSON 一样原生使用（需编译成 JS 或用 protobuf.js）；
5. **二进制流非规范化**：同一份数据可能有多种字节表示，**不能直接对字节流做哈希或逐字节比较来判断相等**。

---

## 8. 版本演进：proto2 → proto3 → Editions

| 版本 | 时间 | 特点 | 现状 |
| --- | --- | --- | --- |
| **proto2** | 2008 | 所有字段显式 optional/required/repeated，支持 default 值、扩展 | 大量存量项目仍在使用 |
| **proto3** | 2016 | 简化语法：默认 singular，去掉 required/default；新增 map、JSON 映射；默认值不编码 | 当前使用最广泛的主流语法 |
| **Editions** | 2023 起 | 统一 proto2/proto3 为同一套语言，用 `edition = "2023";` 声明并按字段定制特性（features） | 新方向，2024、2026 版已发布，protoc 36.x 起完整支持 |

新手建议：**直接从 proto3 学起**（资料最多、使用最广），了解 proto2 的差异即可，关注 Editions 作为未来趋势。

> 版本参考：[Protobuf Version Support](https://protobuf.dev/support/version-support/) —— 当前活跃版本为 protoc 36.x；二进制线格式自诞生以来保持稳定不变。

---

## 9. 常见疑问（FAQ）

**Q1：二进制流看不懂，怎么调试？**
用 `protoc --decode_raw < file.bin` 直接查看，或 `protoc --decode=包名.消息名 文件.proto < file.bin` 按字段名展示。Python 中直接 `print(message)` 也能看到可读形式。

**Q2：能随便给消息加字段吗？**
可以。新增字段使用全新编号，旧程序会跳过不认识的新字段，新程序读旧数据时新字段取默认值。**唯一铁律：编号不可复用。**

**Q3：Protobuf 和 gRPC 是什么关系？**
gRPC 是 Google 的 RPC 框架，默认用 Protobuf 作为接口定义语言和数据格式。可以理解为：Protobuf = "数据类型和接口的合同 + 编码格式"，gRPC = "按合同进行远程调用的网络框架"。二者可独立使用。

**Q4：既然 JSON 好用，为什么还要 Protobuf？**
当数据量大、性能敏感、需要强类型契约（微服务间通信）时，Protobuf 的体积和速度优势是数量级的。建议：**对外/浏览器用 JSON，服务内部用 Protobuf**，两者不是替代关系。

**Q5：为什么反序列化后字段值是默认值？**
可能是发送方没设置该字段（默认值不编码），也可能是传输的字节里确实没有它。需要区分时给字段加 `optional`。

**Q6：可以对序列化结果算 MD5 来比较两条消息吗？**
不行。Protobuf 序列化结果**不保证规范化**（字段顺序、编码方式可能不同），相同数据可能有不同字节表示。比较请用各语言提供的 `==` / `proto.Equal` 等语义比较方法。

**Q7：repeated 字段的 packed 是什么？**
把多个标量值连续编码、只写一次 tag 和总长度的优化方式。proto3 中 repeated 标量字段默认开启，无需手动设置。

---

## 10. 学习路线与官方资源

**推荐学习路线**

1. 读懂一个 `.proto` 文件（本文第 5 章速查足够）；
2. 亲手跑通一次"生成代码 → 序列化 → 反序列化"（本文第 4 章）；
3. 精读官方 [Encoding 文档](https://protobuf.dev/programming-guides/encoding/)，对照本文 3.3 节手工编码几个例子；
4. 学习 gRPC，做一个跨语言的 Hello World 服务；
5. 了解 Editions 与 field presence 等进阶话题。

**官方资源**

- 官方总站：<https://protobuf.dev/>
- 语言指南（proto3）：<https://protobuf.dev/programming-guides/proto3/>
- 编码原理：<https://protobuf.dev/programming-guides/encoding/>
- 各语言入门教程：<https://protobuf.dev/getting-started/>
- 中文文档镜像：<https://protobuf.com.cn/>
- 源码仓库：<https://github.com/protocolbuffers/protobuf>
- gRPC 官方文档：<https://grpc.io/docs/>

---

## 附：核心要点一页纸

1. **是什么**：Google 出的跨语言结构化数据序列化格式 = 契约(IDL) + 紧凑二进制编码 + 代码生成器。
2. **核心流程**：写 `.proto` → `protoc` 生成代码 → 代码里 `Serialize` / `Parse`。
3. **编码原理**：每条字段 = `tag + value`；`tag = (编号<<3)|线类型`；整数用 varint（7 位一组），负数用 ZigZag，字符串/嵌套消息用"长度+内容"；默认值不编码。
4. **兼容性**：字段编号永不复用；新字段可加，旧程序自动跳过未知字段。
5. **怎么选**：性能与契约优先选 Protobuf（配 gRPC），对外可读性优先选 JSON。
6. **入门版本**：直接学 proto3。
