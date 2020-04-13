---
date: 2020-01-08 10:26:29
title: Polymorphism through template concept
subtitle: A boost::iostreams::stream abstraction case-study
layout: post
published: false
tags: ['cpp', 'file system']
---

## TODO

- [ ] [mutipart upload](https://github.com/aws/aws-sdk-cpp/blob/master/aws-cpp-sdk-s3-integration-tests/BucketAndObjectOperationTest.cpp#L938)

- [ ] [generic_stream](https://www.boost.org/doc/libs/1_71_0/libs/iostreams/doc/guide/generic_streams.html)

In turicreate codebase,

```cpp
struct s3url {
  std::string access_key_id;
  std::string secret_key;
  std::string bucket;
  std::string object_name;
  std::string endpoint;
}

```

> A generic input file stream interface that provides unified access to local filesystem, HDFS, S3, in memory files, and can automatically perform gzip decoding.

```cpp
typedef boost::iostreams::stream<fileio_impl::general_fstream_source>
    general_ifstream_base;

class EXPORT general_ifstream : public general_ifstream_base {
  ...
public:
  ...

  size_t file_size();

  size_t get_bytes_read();

  std::shared_ptr<std::istream> get_underlying_stream();

  ...
}
```

I just list the most critical abstractions that `general_ifstream` has. Under the hood, it inherits from a template concept specialization, `boost::iostreams::stream<fileio_impl::general_fstream_source>`.

What is [boost::iostreams::stream](https://www.boost.org/doc/libs/1_71_0/libs/iostreams/doc/guide/generic_streams.html)? Well, it is a **template concept**. Concept is pervail in C++ generic programming. It's a template and it acts like the normal inheritance, where interface class defines the high-level API abstractions for low-level implementations to obey, but it has no abstraction cost, such as virual function call. It's just a contract that the client should provide `APIs` and `category tags` that the concept needs to generate the normal cpp code from template. There might be no compile time warning to tell the client which contract item is not provided because template only generate corresponding contract code if it's used.

```cpp
namespace boost { namespace iostreams {
//
// Template name: stream.
// Description: A iostream which reads from and writes to an instance of a
//      designated device type.
// Template parameters:
//      Device - A device type.
//      Alloc - The allocator type.
//
template< typename Device,
          typename Tr =
              BOOST_IOSTREAMS_CHAR_TRAITS(
                  BOOST_DEDUCED_TYPENAME char_type_of<Device>::type
              ),
          typename Alloc =
              std::allocator<
                  BOOST_DEDUCED_TYPENAME char_type_of<Device>::type
              > >
struct stream : detail::stream_base<Device, Tr, Alloc> {
public:
    ...
    stream() { }
    BOOST_IOSTREAMS_FORWARD( stream, open_impl, Device,
                             BOOST_IOSTREAMS_PUSH_PARAMS,
                             BOOST_IOSTREAMS_PUSH_ARGS )
    bool is_open() const { return this->member.is_open(); }
    void close() { this->member.close(); }
    bool auto_close() const { return this->member.auto_close(); }
    void set_auto_close(bool close) { this->member.set_auto_close(close); }
    bool strict_sync() { return this->member.strict_sync(); }
    Device& operator*() { return *this->member; }
    Device* operator->() { return &*this->member; }
    Device* component() { return this->member.component(); }
private:
    void open_impl(const Device& dev BOOST_IOSTREAMS_PUSH_PARAMS()) // For forwarding.
    {
        this->clear();
        this->member.open(dev BOOST_IOSTREAMS_PUSH_ARGS());
    }
};
```


```cpp
class general_fstream_source {
  /// The source device must be copyable; thus the shared_ptr.
  std::shared_ptr<union_fstream> in_file;
 public:
  typedef char        char_type;
  struct category: public boost::iostreams::device_tag,
    boost::iostreams::closable_tag,
    boost::iostreams::multichar_tag,
    boost::iostreams::input_seekable,
    boost::iostreams::optimally_buffered_tag {};

```

The `general_fstream_source` just wraps `union_fstream`, which is the underlying stream device interface. Let's follow through the abstraction chain,

```cpp
class union_fstream {

 public:
  enum stream_type {HDFS, STD, CACHE};
  /**
   * Constructs a union fstream from a filename. Based on the filename
   * (whether it begins with hdfs://, or cache://)
   * an appropriate stream type (HDFS, STD, or CACHE) is created.
   *
   * Throws an std::io_base_failure exception if fail to construct the stream.
   */
  union_fstream(std::string url,
                std::ios_base::openmode mode = std::ios_base::in | std::ios_base::out,
                std::string proxy="");

  /// Destructor
  ~union_fstream();

  /// Returns the current stream type. Whether it is a HDFS, STD, or cache stream
  stream_type get_type() const;

  std::shared_ptr<std::istream> get_istream();

  std::shared_ptr<std::ostream> get_ostream();
  ...

  /**
   * Returns the file size of the opened file.
   * Returns (size_t)(-1) if there is no file opened, or if there is an
   * error obtaining the file size.
   */
  size_t file_size();

 private:
  union_fstream(const union_fstream& other) = delete;

 private:

  stream_type type;
  std::string url;
  size_t m_file_size = (size_t)(-1);

  std::shared_ptr<std::istream> input_stream;
  std::shared_ptr<std::ostream> output_stream;
  // Hold the input stream from cache or s3 stream.
  std::shared_ptr<std::istream> original_input_stream_handle;
};
```

Now everything is clear, this `union_fstream` is a factor for different types of file system, such as s3, hdfs, local file, and etc. It returns `std::shared_ptr<std::istream>` or `std::shared_ptr<std::ostream>` for upper abstractions.

Let's have a closer look at its factor method, which is the constructor,

```cpp
union_fstream::union_fstream(std::string url, std::ios_base::openmode mode,
                             std::string proxy)
  ...
  else if (fileio::get_protocol(url) == "s3") {
    // the S3 file type currently works by download/uploading a local file
    // i.e. the s3_stream simply remaps a local file stream
    type = STD;
    if (is_output_stream) {
      output_stream = std::make_shared<s3_fstream>(url, /* write */ true);
    } else {
      auto s3stream = std::make_shared<s3_fstream>(url, false);
      input_stream = (*s3stream)->get_underlying_stream();
      if (input_stream == nullptr) input_stream = s3stream;
      m_file_size = (*s3stream)->file_size();
      original_input_stream_handle =
          std::static_pointer_cast<std::istream>(s3stream);
    }
  ...
```

If the url starts with `s3://`, it will use `s3_fstream` to fetch the file designated by the url. After further reading the source code, I figure out that it is wrapped by `boost::iostreams::stream` concept. Therefore, let's dig into the underlying `s3_device`, which provides the concrete implementaion for s3 file streaming.

```cpp
class s3_device {
 public: // boost iostream concepts
  typedef char                                          char_type;
  struct category :
      public boost::iostreams::device_tag,
      public boost::iostreams::multichar_tag,
      public boost::iostreams::closable_tag,
      public boost::iostreams::bidirectional_seekable { };
  // while this claims to be bidirectional_seekable, that is not true
  // it is only read seekable. Will fail when seeking on write
 private:
  std::string remote_fname;
  std::shared_ptr<dmlc::io::S3FileSystem> m_s3fs;
  std::shared_ptr<dmlc::Stream> m_write_stream;
  std::shared_ptr<dmlc::SeekStream> m_read_stream;
  size_t m_filesize = (size_t)(-1);
 public:
  ...

  s3_device(const std::string& filename, const bool write = false);

  // Because the device has bidirectional tag, close will be called
  // twice, one with the std::ios_base::in, followed by out.
  // Only close the file when the close tag matches the actual file type.
  void close(std::ios_base::openmode mode = std::ios_base::openmode());

  /** the optimal buffer size is 0. */
  inline std::streamsize optimal_buffer_size() const { return 0; }

  std::streamsize read(char* strm_ptr, std::streamsize n);

  std::streamsize write(const char* strm_ptr, std::streamsize n);

  bool good() const;

  /**
   * Seeks to a different location.
   */
  std::streampos seek(std::streamoff off,
                      std::ios_base::seekdir way,
                      std::ios_base::openmode);

  size_t file_size() const;

  std::shared_ptr<std::istream> get_underlying_stream();
  std::string m_filename;
}; // end of s3 device

typedef boost::iostreams::stream<s3_device> raw_s3_fstream;
typedef boost::iostreams::stream<read_caching_device<s3_device>> s3_fstream;
```
