---
tags:
  - CMake
---

## 生成compile_commands.json
compile_commands.json是一种编译数据库文件，通过他可以实现自动跳转等功能。
```sh
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON .
```