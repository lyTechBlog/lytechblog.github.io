---
title: 使用日志库spdlog
sidebar: mydoc_sidebar
permalink: spdlog.html
folder: mydoc
---
## 结论
使用spdlog，其是目前c++日志库中，性能最好的。

### 结论延伸（建议读者学习spdlog后查看）
- 性能：使用2个线程时，每条日志1kb，可以达到16G/min。这个日志量，对大部分业务是足够的。
- spdlog瓶颈：spdlog的sink_it（作用是format字符串、调用std::fwrite），是加锁的，也就是无论初始化spdlog的线程池初始化多少个，sink_it效率和单线程无异，而std::fwrite就是其性能瓶颈。所以，想要更快的性能的写日志方式是可行的。通过改spdlog源码，是可以实现的，这个我之后会再开一篇blog记录。
- 新增功能：spdlog不支持分钟级别数据轮转，且保持当前写文件的名称不变的需求，在文末，我贴上我的sink源码。测试源码：

## 环境说明
代码运行在32核linux机器，g++版本4.85，系统版本centos7。磁盘是挂载的云盘，单线程实际写入性能约20G/min

## 运行代码
初始化日志
```xml
#define LOG_DEBUG(...) logger->debug(__VA_ARGS__)
#define LOG_INFO(...) logger->info(__VA_ARGS__)
#define LOG_WARN(...) logger->warn(__VA_ARGS__)
#define LOG_ERROR(...) logger->error(__VA_ARGS__)

namespace userProfile {

static std::shared_ptr<spdlog::async_logger> logger = nullptr;

void initLogger(std::shared_ptr<const ImmutableConf> conf) {
  auto dist_sink = std::make_shared<spdlog::sinks::dist_sink_st>();
  auto daily_file = std::make_shared<spdlog::sinks::mins_rotating_sink_mt> (conf->get_stdlog_path(), conf->get_file_interval_mins(), false);
  dist_sink->add_sink(daily_file);
  
  spdlog::init_thread_pool(1000000, 2);
  logger = std::make_shared<spdlog::async_logger>("as", dist_sink, spdlog::thread_pool(), spdlog::async_overflow_policy::overrun_oldest);
  spdlog::flush_on(spdlog::level::info);
  return;
}
```

分钟级别日志的三方库
```json
#pragma once

/**
 * @brief 扩展开源库spdlog的写日志功能，支持按指定时间间隔，轮转写日志
 * 如:  日志文件名前缀为 a.log，文件存储间隔60s，
 *      那么若当前时间为2023-04-19-15-28
 *      则当前正在写的文件为 a.log
 *      而上一分钟写的文件名 a.log_2023-04-19-15-27
 * 
 * 原理： 时间达到时，阻塞写，将a.log 改名为 a.log_2023-04-19-15-27, 然后新建a继续写
 */

#include <spdlog/common.h>
#include <spdlog/details/file_helper.h>
#include <spdlog/details/null_mutex.h>
#include <spdlog/fmt/fmt.h>

#include <chrono>
#include <mutex>
#include <sys/stat.h>

namespace spdlog {
namespace sinks {
static const int MAX_SUFFIX_FILE_NUM = 100;

struct mins_rotating_filename_calculator
{
  static filename_t calc_filename(const filename_t &filename, const tm &now_tm)
  {
    filename_t basename, ext;
    std::tie(basename, ext) = details::file_helper::split_by_extension(filename);
    return fmt_lib::format(SPDLOG_FILENAME_T("{}_{:04d}-{:02d}-{:02d}-{:02d}-{}"), basename, now_tm.tm_year + 1900, now_tm.tm_mon + 1,
        now_tm.tm_mday, now_tm.tm_hour, now_tm.tm_min, ext);
  }
};

template<typename Mutex, typename FileNameCalc = mins_rotating_filename_calculator>
class mins_rotating_sink final : public base_sink<Mutex>
{
public:
  mins_rotating_sink( filename_t base_filename, 
                        int interval_mins,
                        bool truncate = false, 
                        const file_event_handlers &event_handlers = {})
    : base_filename_(base_filename),
      truncate_(truncate),
      file_helper_(event_handlers),
      interval_mins_(interval_mins)

  {
    auto now = log_clock::now();
    file_helper_.open(base_filename_, truncate_);
    rotation_tp_ = next_rotation_tp_();
    remove_init_file_ = file_helper_.size() == 0;
  }

  filename_t filename()
  {
      std::lock_guard<Mutex> lock(base_sink<Mutex>::mutex_);
      return file_helper_.filename();
  }

protected:
  void sink_it_(const details::log_msg &msg) override
  {
      auto time = msg.time;
      bool should_rotate = time >= rotation_tp_;
      if (should_rotate)
      {
          flush_();
          file_helper_.close();
          if (remove_init_file_)
          {
              details::os::remove(file_helper_.filename());
          }
          auto filename = FileNameCalc::calc_filename(base_filename_, now_tm(time));

          // 当应用重启时，move文件为包含日期后缀时，会判断该后缀文件时候存在
          int suffix = 0;
          struct stat buffer;
          filename_t new_filename = filename;
          while (suffix < MAX_SUFFIX_FILE_NUM) {
            if (suffix != 0) {
              filename_t new_filename = fmt::format("{}.{}", filename, suffix);
            }
            // 添加like or unlikely帮助分支预测

            if (stat(new_filename.c_str(), &buffer) != 0) {
              std::rename(base_filename_.c_str(), new_filename.c_str());
              break;
            }
            suffix++;
          }
          file_helper_.open(base_filename_, truncate_);
          rotation_tp_ = next_rotation_tp_();
      }
      remove_init_file_ = false;
      memory_buf_t formatted;
      base_sink<Mutex>::formatter_->format(msg, formatted);
      file_helper_.write(formatted);
      flush_();
  }

  void flush_() override
  {
      file_helper_.flush();
  }

private:
  tm now_tm(log_clock::time_point tp)
  {
      time_t tnow = log_clock::to_time_t(tp);
      return spdlog::details::os::localtime(tnow);
  }

  log_clock::time_point next_rotation_tp_()
  {
      auto now = log_clock::now();
      tm date = now_tm(now);
      date.tm_sec = 0;
      auto rotation_time = log_clock::from_time_t(std::mktime(&date));
      if (rotation_time > now)
      {
          return rotation_time;
      }
      return {rotation_time + std::chrono::minutes(interval_mins_)};
  }
private:
    filename_t base_filename_;
    log_clock::time_point rotation_tp_;
    details::file_helper file_helper_;
    int interval_mins_;
    bool truncate_;
    bool remove_init_file_;
};

using mins_rotating_sink_mt = mins_rotating_sink<std::mutex>;
using mins_rotating_sink_st = mins_rotating_sink<details::null_mutex>;

} // namespace sinks
} // namespace spdlog
```

