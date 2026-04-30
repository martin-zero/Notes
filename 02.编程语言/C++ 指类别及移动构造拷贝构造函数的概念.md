# 值类别
C++11中，值类别可以分为**左值（lvalue）、将亡值（xvalue）以及纯右值（prvalue）**，值类别用来精确描述表达式的属性，从而支持:
- 移动语义（move semantics）
- 完美转发（perfect forwarding）
- 减少不必要拷贝

## 左值(lvalue)
