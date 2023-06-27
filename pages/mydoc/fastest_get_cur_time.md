---
title: 使用"clock_gettime"或者"gettimeofday"获取当前时间
sidebar: mydoc_sidebar
permalink: fastest_get_cur_time.html
folder: mydoc
---
 
## 结论
- c++11中获取unix时间最快的方式是clock_gettime和gettimeofday函数，每个线程每秒获取约4千万次。chrono库获取约1千万次

### 结论延伸
- 单线程和多线程对时间获取的性能无影响。因为时间数据已经做到读写分离。由于环境差异，会导致时间有差别，快速测试，可使用代码[https://github.com/lyTechBlog/blog_example_code/tree/example/get_time](https://github.com/lyTechBlog/blog_example_code/tree/example/get_time) <br/>
- 内核获取时间，是通过定期NTP网络时间同步+自身的时钟滴答数/每秒滴答数
- 分布式系统中，任何一个机器的时间不是一个绝对正确的值，而是一个置信度区间

## 环境说明
代码运行在32核linux机器，g++版本4.85，系统版本centos7

## 测试结果
{% include image.html file="fastest_get_cur_time_pic_1.png" caption="" %}

## 测试使用代码：
```cpp
#include <iostream>
#include <chrono>
#include <ctime>
#include <sys/time.h>
#include <thread>
#include <vector>

using namespace std;
using namespace std::chrono;

const int TEST_TIMES = 1000 * 1000 * 10;

long long getCurrentTimeByClockGetTime() {
    struct timespec spec;
    clock_gettime(CLOCK_REALTIME, &spec);
    return spec.tv_sec * 1000LL + spec.tv_nsec / 1000000;
}

long long getCurrentTimeByGetTimeOfDay() {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000LL + tv.tv_usec / 1000;
}

long long getCurrentTimeByChrono() {
    return duration_cast<milliseconds>(high_resolution_clock::now().time_since_epoch()).count();
}

void testClockGetTime() {
    long long start = getCurrentTimeByChrono();
    for (int i = 0; i < TEST_TIMES; ++i) {
        getCurrentTimeByClockGetTime();
    }
    long long end = getCurrentTimeByChrono();
    cout << "Using clock_gettime in thread " << this_thread::get_id() << ": " << end - start << " ms\n";
}

void testGetTimeOfDay() {
    long long start = getCurrentTimeByChrono();
    for (int i = 0; i < TEST_TIMES; ++i) {
        getCurrentTimeByGetTimeOfDay();
    }
    long long end = getCurrentTimeByChrono();
    cout << "Using gettimeofday in thread " << this_thread::get_id() << ": " << end - start << " ms\n";
}

void testChrono() {
    long long start = getCurrentTimeByChrono();
    for (int i = 0; i < TEST_TIMES; ++i) {
        getCurrentTimeByChrono();
    }
    long long end = getCurrentTimeByChrono();
    cout << "Using chrono::high_resolution_clock in thread " << this_thread::get_id() << ": " << end - start << " ms\n";
}

int testMultiThread(int thread_num) {
    vector<thread> threads;

    for (int i = 0; i < thread_num; ++i) {
        threads.emplace_back(testChrono);
    }
    for (auto& t : threads) {
        t.join();
    }

    threads.clear();
    for (int i = 0; i < thread_num; ++i) {
        threads.emplace_back(testGetTimeOfDay);
    }
    for (auto& t : threads) {
        t.join();
    }

    threads.clear();
    for (int i = 0; i < thread_num; ++i) {
        threads.emplace_back(testClockGetTime);
    }
    for (auto& t : threads) {
        t.join();
    }

    return 0;
}



int main() {
    testMultiThread(1);
    testMultiThread(10);
}
```
