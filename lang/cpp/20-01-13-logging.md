---
date: 2020-01-13 15:03:40
title: logging config
subtitle:
layout: post
published: false
tags: ['cpp', 'file system', 'logging']
---


logger path, mutex based, and everything is written to underlying file. Buffering is explicitly enabled by `_logbuf`.

```cpp
#define logprogress(fmt,...) logger(LOG_PROGRESS, fmt, ##__VA_ARGS__)
#define logprogress_stream logstream(LOG_PROGRESS)

#define logger(lvl, fmt, ...)                                                  \
  (log_dispatch<(lvl >= OUTPUTLEVEL)>::exec(lvl, __FILE__, __func__, __LINE__, \
                                            fmt, ##__VA_ARGS__))

...

template <>
struct log_dispatch<true> {
  inline static void exec(int loglevel,const char* file,const char* function,
                          int line,const char* fmt, ... ) {
    va_list argp;
    va_start(argp, fmt);
    global_logger()._log(loglevel, file, function, line, fmt, argp);
    va_end(argp);
    if(loglevel == LOG_FATAL) {
      __print_back_trace();
      TURI_LOGGER_FAIL_METHOD("LOG_FATAL encountered");
    }
  }
};

void file_logger::_log(int lineloglevel,const char* file,const char* function,
                       size_t line,const char* fmt, va_list ap ) {
  // if the logger level fits
  if (lineloglevel >= log_level){
    // get just the filename. this line found on a forum on line.
    // claims to be from google.
    file = ((strrchr(file, '/') ? : file- 1) + 1);

    char str[1024];

    // write the actual header (only show file in debug build)
#ifndef NDEBUG
    int head_bytes_written = snprintf(str,1024, "%s%s(%s:%d): ",
                                  messages[lineloglevel],file,function,static_cast<int>(line));
#else
    int head_bytes_written = snprintf(str,1024, "%s(%s:%d): ",
                                  messages[lineloglevel],function,static_cast<int>(line));
#endif
    // write the actual logger

    int byteswritten = head_bytes_written +
        vsnprintf(str + head_bytes_written, 1024 - head_bytes_written,fmt,ap);

    str[byteswritten] = '\n';
    str[byteswritten+1] = 0;
    // write the output
    if (has_callback[lineloglevel]) {
      pthread_mutex_lock(&mut);
      if (callback[lineloglevel]) {
        // only emit the message not the header
        callback[lineloglevel](lineloglevel,
                               str + head_bytes_written,
                               byteswritten - head_bytes_written);
      }
      pthread_mutex_unlock(&mut);
    }
    _lograw(lineloglevel, str, byteswritten);
  }
}

```

thread local path to write the stream and flush the buffered stream once.

```cpp
#define logstream(lvl)                                                      \
  if (lvl >= global_logger().get_log_level())                               \
  (log_stream_dispatch<(lvl >= OUTPUTLEVEL)>::exec(lvl, __FILE__, __func__, \
                                                   __LINE__))
...
template <>
struct log_stream_dispatch<true> {
  inline static file_logger& exec(int lineloglevel,const char* file,const char* function, int line, bool do_start = true) {
    // First see if there is an interupt waiting.  This is a convenient place that a lot of people call.
    if(cppipc::must_cancel()) {
      log_and_throw("Canceled by user.");
    }


    return global_logger().start_stream(lineloglevel, file, function, line, do_start);
  }
};

namespace logger_impl {
struct streambuff_tls_entry {
  std::stringstream streambuffer;
  bool streamactive;
  size_t header_len;
  int loglevel;
};


file_logger& file_logger::start_stream(int lineloglevel,const char* file,
                                       const char* function, int line, bool do_start) {
  // get the stream buffer
  logger_impl::streambuff_tls_entry* streambufentry =
        reinterpret_cast<logger_impl::streambuff_tls_entry*>(
                              pthread_getspecific(streambuffkey));
  // create the key if it doesn't exist
  if (streambufentry == NULL) {
    streambufentry = new logger_impl::streambuff_tls_entry;
    pthread_setspecific(streambuffkey, streambufentry);
  }
  if (streambufentry->streambuffer.str().length() > 0) {
    (*this) << std::endl;
  }
  std::stringstream& streambuffer = streambufentry->streambuffer;
  bool& streamactive = streambufentry->streamactive;
  size_t& header_len = streambufentry->header_len;
  streambufentry->loglevel = lineloglevel;

  if (lineloglevel >= log_level){
    // get the stream buffer
    // if do not start the stream, just quit
    if (do_start == false) {
      streamactive = false;
      return *this;
    }

#ifndef NDEBUG
    file = ((strrchr(file, '/') ? : file- 1) + 1);

    if (streambuffer.str().length() == 0) {
      streambuffer << int(time(NULL)) << " : " << messages[lineloglevel] << file
                   << "(" << function << ":" <<line<<"): ";
    }
#else
    if (streambuffer.str().length() == 0) {
      streambuffer << int(time(NULL)) << " : " << messages[lineloglevel] << "(" << function << ":" <<line<<"): ";
    }
#endif
    streamactive = true;
    streamloglevel = lineloglevel;
    header_len = streambuffer.tellp();
  } else {
    streamactive = false;
  }
  return *this;
}

  /**
   * Streams a value into the logger.
   */
  template <typename T>
  file_logger& operator<<(T a) {
    // get the stream buffer
    logger_impl::streambuff_tls_entry* streambufentry = reinterpret_cast<logger_impl::streambuff_tls_entry*>(
                                          pthread_getspecific(streambuffkey));
    if (streambufentry != NULL) {
      std::stringstream& streambuffer = streambufentry->streambuffer;
      bool& streamactive = streambufentry->streamactive;

      if (streamactive) {
        streambuffer << a;
      }
    }
    return *this;
  }

```

All 2 paths converge on writing buffer to log file.

[pthread_getspecific](https://linux.die.net/man/3/pthread_getspecific) is used in pair with `pthead_setspecific`.

```cpp
void file_logger::_lograw(int lineloglevel, const char* buf, size_t len) {
  pthread_mutex_lock(&mut);
  if (fout.good()) {
    fout.write(buf,len);
    fout.flush();
  }
  pthread_mutex_unlock(&mut);
  if (log_to_console || log_to_stderr) {
#ifdef COLOROUTPUT

    pthread_mutex_lock(&mut);
    if (lineloglevel == LOG_FATAL) {
      textcolor(stderr, TEXTCOLOR_BRIGHT, TEXTCOLOR_RED);
    } else if (lineloglevel == LOG_ERROR) {
      textcolor(log_to_stderr ? stderr : stdout, TEXTCOLOR_BRIGHT,
                TEXTCOLOR_RED);
    } else if (lineloglevel == LOG_WARNING) {
      textcolor(log_to_stderr ? stderr : stdout, TEXTCOLOR_BRIGHT,
                TEXTCOLOR_MAGENTA);
    } else if (lineloglevel == LOG_DEBUG) {
      textcolor(log_to_stderr ? stderr : stdout, TEXTCOLOR_BRIGHT,
                TEXTCOLOR_YELLOW);
    } else if (lineloglevel == LOG_EMPH) {
      textcolor(log_to_stderr ? stderr : stdout, TEXTCOLOR_BRIGHT,
                TEXTCOLOR_GREEN);
    }
#endif
    if (lineloglevel >= LOG_FATAL) {
      std::cerr.write(buf, len);
    } else {
      (log_to_stderr ? std::cerr : std::cout).write(buf, len);
    }
#ifdef COLOROUTPUT

    pthread_mutex_unlock(&mut);
    if (lineloglevel >= LOG_FATAL) {
      reset_color(stderr);
    } else {
      reset_color(log_to_stderr ? stderr : stdout);
    }
#endif
  }
}
```

server disables it,

```cpp
EXPORT void start_server(const unity_server_options& server_options,
                         const unity_server_initializer& server_initializer) {
  std::lock_guard<mutex> server_start_lg(_server_start_lock);
  namespace fs = boost::filesystem;
  global_logger().set_log_level(LOG_PROGRESS);
  global_logger().set_log_to_console(false);
  if(SERVER) {
    logstream(LOG_ERROR) << "Unity server initialized twice." << std::endl;
    return;
  }
  SERVER = new unity_server(server_options);
  SERVER->start(server_initializer);
}
```


log progress has special call back settings on python layer,

```cpp
EXPORT void set_log_progress_callback( void (*callback)(const std::string&) ) {
  if (SERVER) {
    SERVER->set_log_progress_callback(callback);
  }
}

// in unity_server.cpp
void unity_server::set_log_progress_callback(progress_callback_type callback) {
  if (callback == nullptr) {
    log_progress_callback = nullptr;
    global_logger().add_observer(LOG_PROGRESS, NULL);
  } else {
    log_progress_callback = callback;
    global_logger().add_observer(
        LOG_PROGRESS,
        [=](int lineloglevel, const char* buf, size_t len){
          this->log_queue.enqueue(std::string(buf, len));
        });
  }
}

// unity_server.hpp
  turi::thread log_thread;
  blocking_queue<std::string> log_queue;
```

```python
# Cython code
    def set_log_progress(self, enable):
        if enable:
            set_log_progress_callback(print_status)
            self._log_progress_enabled = True
        else:
            set_log_progress(False)
            self._log_progress_enabled = False

```

```python
def print_callback(val):
    """
    Internal function.
    This function is called via a call back returning from IPC to Cython
    to Python. It tries to perform incremental printing to IPython Notebook or
    Jupyter Notebook and when all else fails, just prints locally.
    """
    success = False
    try:
        # for reasons I cannot fathom, regular printing, even directly
        # to io.stdout does not work.
        # I have to intrude rather deep into IPython to make it behave
        if have_ipython:
            if InteractiveShell.initialized():
                IPython.display.publish_display_data({'text/plain':val,'text/html':'<pre>' + val + '</pre>'})
                success = True
    except:
        pass

    if not success:
        print(val)
        sys.stdout.flush()
```

in `_main.py` when server starts,

```python
    # set the verbose level back to default
    glconnect.get_server().set_log_progress(True)
```