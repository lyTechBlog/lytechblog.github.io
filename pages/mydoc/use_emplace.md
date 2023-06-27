---
title: 使用"emplace"类函数向容器中添加数据
sidebar: mydoc_sidebar
permalink: use_emplace.html
folder: mydoc
---
 
## 结论

- vector使用emplace_back替换push_back，map使用emplace替换insert

### 结论补充说明：

- emplace的性能总是相当或略好于push/insert。使用代码：[https://github.com/lyTechBlog/blog_example_code/tree/example/emplace_back](https://github.com/lyTechBlog/blog_example_code/tree/example/emplace_back)

## 原理

- 原理上来说，push是构造函数+移动赋值函数。emplace只调用了构造函数

   - pusk类函数的一种实现
```cpp
void push(const T& value) {
   if (size_ == capacity_) {
       reserve(2 * capacity_); // double the capacity of the vector
   }
   data_[size_] = value; // insert the new element at the end of the vector
   size_++; // increment the size of the vector
}
```

   - emplace类函数的一种实现
```cpp
template <typename... Args>
void emplace(Args&&... args) {
   if (size_ == capacity_) {
       reserve(2 * capacity_); // double the capacity of the vector
   }
   new (&data_[size_]) T(std::forward<Args>(args)...); // construct the new element in place
   size_++; // increment the size of the vector
}
```

## 测试说明：
<br />-O0编译，emplace好于push/insert<br />
{% include image.html file="use_emplace_1.png" caption="" %}
<br />-O2编译，emplace好于push/insert<br />
{% include image.html file="use_emplace_2.png" caption="" %}

测试源码

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <typeinfo>
#include <map>

using namespace std;
using namespace std::chrono;

template <typename T>
void test_vec(const T& t, int add_times = 1000 * 1000) {
  cout << "*** vector type: " << typeid(T).name() << "\n";

  // 测试emplace_back
  vector<T> v1;
  auto start1 = high_resolution_clock::now();  // 记录开始时间
  for (int i = 0; i < add_times; i++) {
      v1.emplace_back(t);
  }
  auto end1 = high_resolution_clock::now();  // 记录结束时间
  auto duration1 = duration_cast<milliseconds>(end1 - start1);
  cout << "emplace_back: " << duration1.count() << " milliseconds" << "\n";
  
  // 测试push_back
  vector<T> v2;

  auto start2 = high_resolution_clock::now();  // 记录开始时间
  for (int i = 0; i < add_times; i++) {
      v2.push_back(T(t));
  }
  auto end2 = high_resolution_clock::now();  // 记录结束时间
  auto duration2 = duration_cast<milliseconds>(end2 - start2);
  cout << "push_back: " << duration2.count() << " milliseconds" << "\n";
  cout << "\n";
}

template <typename T>
void test_map(const T& t, int add_times = 1000 * 1000) {
    std::cout << "*** map type: " << typeid(T).name() << "\n";

    // 测试 emplace
    std::map<T, T> m2;
    auto start2 = std::chrono::high_resolution_clock::now();   // 记录开始时间
    for (int i = 0; i < add_times; ++i) {
        m2.emplace(i, i);
    }
    auto end2 = std::chrono::high_resolution_clock::now();   // 记录结束时间
    auto duration2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2);
    std::cout << "emplace: " << duration2.count() << " milliseconds" << "\n";

    // 测试 insert
    std::map<T, T> m1;
    auto start1 = std::chrono::high_resolution_clock::now();   // 记录开始时间
    for (int i = 0; i < add_times; ++i) {
        m1.insert(std::make_pair(i, i));
    }
    auto end1 = std::chrono::high_resolution_clock::now();   // 记录结束时间
    auto duration1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1);
    std::cout << "insert: " << duration1.count() << " milliseconds" << "\n";
    cout << "\n";
}

template <>
void test_map<std::string>(const std::string& t, int add_times) {
    std::cout << "*** map type: " << typeid(std::string).name() << "\n";

    // 测试 emplace
    std::map<std::string, std::string> m2;
    auto start2 = std::chrono::high_resolution_clock::now();   // 记录开始时间
    for (int i = 0; i < add_times; ++i) {
        m2.emplace(std::to_string(i), std::to_string(i));
    }
    auto end2 = std::chrono::high_resolution_clock::now();   // 记录结束时间
    auto duration2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2);
    std::cout << "emplace: " << duration2.count() << " milliseconds" << "\n";

    // 测试 insert
    std::map<std::string, std::string> m1;
    auto start1 = std::chrono::high_resolution_clock::now();   // 记录开始时间
    for (int i = 0; i < add_times; ++i) {
        m1.insert(std::make_pair(std::to_string(i), std::to_string(i)));
    }
    auto end1 = std::chrono::high_resolution_clock::now();   // 记录结束时间
    auto duration1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1);
    std::cout << "insert: " << duration1.count() << " milliseconds" << "\n";
    cout << "\n";
}

int main() {
    std::string str(100, 'a'); ;
    test_vec(1);
    test_vec(str);

    test_map(1);
    test_map(str);

    return 0;
}
```
