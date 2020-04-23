---
layout: post
title: "How AWS CPP SDK handles retry on S3 errors?"
subtitle: "An alternative documentation for retry strategy of AWS CPP SDK"
published: true
bigimg: /img/color_path.jpg
tag: ["aws", "cpp"]
categories: ["aws", "cpp"]
date: 2020-04-12 21:33:32
---

## Summary

AWS SDK will do retry for you under 3 conditions

1. client network failures. Usually, requests are not made to the server.

2. deciding from Http response code when the response body from server side is empty. The logic is in [IsRetryableHttpResponseCode][1]. It will retry on server (5xx), throttling (599, bandwidth-limited), and some client (4xx) errors.

3. Http response body contains non-empty message. It will try to marshall (map) the exception name found in the message to bool results, where `true` means should retry. For instance, exception name `SlowDownException` (HTTP code 429) corresponds to `true`. Regarding different services, it may use different client implementation for marshaling. For S3, it uses this [marshaling](https://github.com/aws/aws-sdk-cpp/blob/master/aws-cpp-sdk-s3/source/S3ErrorMarshaller.cpp), which marshalls S3 specific exception name and then fall back to base strategy offered by [CoreErrors](https://github.com/aws/aws-sdk-cpp/blob/master/aws-cpp-sdk-core/source/client/CoreErrors.cpp).

The [default retry strategy](https://github.com/aws/aws-sdk-cpp/blob/master/aws-cpp-sdk-core/source/client/DefaultRetryStrategy.cpp) use 3 for max retry time. But you can customize your own max retry time and [retry strategy](https://github.com/aws/aws-sdk-cpp/blob/master/aws-cpp-sdk-core/source/client/SpecifiedRetryableErrorsRetryStrategy.cpp) based on specific error messages.

All in all, you can customize retry strategy by providing a list of [error names][2], or check `ShouldRetry` and `GetErrorType` and then make decision from information provided by error instances.

```cpp
do {
    auto outcome = client.ListObjectsV2(request);
    if (outcome.IsSuccess()) {
        // blah
    } else {
        auto error = outcome.GetError();
        if (error.ShouldRetry()) {
            // SDK already retried, no need to retry on our own
            // error report or logging
            break;
        }
        else
        {
            // if it's unknown error (non-standard AWS error), we may need
            // to check the exception name or exception message.
            if (error.GetErrorType() == Aws::S3::S3Errors::UNKNOWN) {
                // custom retry
                // check http response code?
                // check error name or error message?
                n_retry++;
                ...
            } else {
                // respect AWS's decision on retry
            }
        }
while (n_retry < 3);
```

In most cases, you don't need to explicitly retry. But in rare cases, such as private S3 proxy returns non-standard AWS errors (explained below), you need to explicitly retry on your own judgement.

---

## Case Study by Code

So, you may ask how Aws CPP SDK handles retry on s3 request errors? I read the [documentation](https://docs.aws.amazon.com/general/latest/gr/api-retries.html), it says AWS will perform retry for me. However, based on my observation, it doesn't retry on S3 errors.

Let's go through the example with HTTP code `429`. The internal S3 service at my work has rate limitations on `ListObjectsV2`, and once there are too many requests on the same key within a short period, it will deny the request by returning code 429.

Thus, I decided to provide a custom retry logic. I happened to see there's a `ShouldRetry` method on s3 error instance, and why not have a shot? AWS might already have some internal retry logic prepared for us, just like [CoreErrors][0].

The code is like,

```cpp
do {
    auto outcome = client.ListObjectsV2(request);
    if (outcome.IsSuccess()) {
        // blah
    } else {
        auto error = outcome.GetError();

        if (error.ShouldRetry()) {
            // custom retry
            n_retry++;

            ...
while (n_retry < 3);
```

But `error.ShouldRetry` **always** returns false from S3 client, which means I probably need to customize retry strategy for S3 requests.

To check what it is, go through stack trace from lldb,

```bash
frame #0: 0x0000000105228ed2 libunity_shared_for_testing.dylib`Aws::S3::S3ErrorMapper::GetErrorForName(errorName="429") at S3Errors.cpp:56:12
frame #1: 0x0000000105226c36 libunity_shared_for_testing.dylib`Aws::Client::S3ErrorMarshaller::FindErrorByName(this=0x0000000126d29e38, errorName="429") const at S3ErrorMarshaller.cpp:25:32
frame #2: 0x0000000104e9d4ed libunity_shared_for_testing.dylib`Aws::Client::AWSErrorMarshaller::Marshall(this=0x0000000126d29e38, exceptionName=0x00007ffeefbf7dd0, message=0x00007ffeefbf7da0) const at AWSErrorMarshaller.cpp:137:34
frame #3: 0x0000000104e9de9a libunity_shared_for_testing.dylib`Aws::Client::XmlErrorMarshaller::Marshall(this=0x0000000126d29e38, httpResponse=0x0000000126d2a848) const at AWSErrorMarshaller.cpp:94:25
frame #4: 0x0000000104e937da libunity_shared_for_testing.dylib`Aws::Client::AWSXMLClient::BuildAWSError(this=0x00007ffeefbfb0f0, httpResponse=std::__1::shared_ptr<Aws::Http::HttpResponse>::element_type @ 0x0000000126d2a848 strong=1 weak=1) const at AWSClient.cpp:901:39
frame #5: 0x0000000104e8af90 libunity_shared_for_testing.dylib`Aws::Client::AWSClient::AttemptOneRequest(this=0x00007ffeefbfb0f0, httpRequest=std::__1::shared_ptr<Aws::Http::HttpRequest>::element_type @ 0x0000000126d2a300 strong=2 weak=1, request=0x00007ffeefbfaed0, signerName="SignatureV4", signerRegionOverride="store-test") const at AWSClient.cpp:319:20
frame #6: 0x0000000104e89b57 libunity_shared_for_testing.dylib`Aws::Client::AWSClient::AttemptExhaustively(this=0x00007ffeefbfb0f0, uri=0x00007ffeefbfa058, request=0x00007ffeefbfaed0, method=HTTP_GET, signerName="SignatureV4", signerRegionOverride="store-test") const at AWSClient.cpp:184:19
frame #7: 0x0000000104e92580 libunity_shared_for_testing.dylib`Aws::Client::AWSXMLClient::MakeRequest(this=0x00007ffeefbfb0f0, uri=0x00007ffeefbfa058, request=0x00007ffeefbfaed0, method=HTTP_GET, signerName="SignatureV4", signerRegionOverride="store-test") const at AWSClient.cpp:827:48
frame #8: 0x0000000104fe43ec libunity_shared_for_testing.dylib`Aws::S3::S3Client::ListObjectsV2(this=0x00007ffeefbfb0f0, request=0x00007ffeefbfaed0) const at S3Client.cpp:2824:24
```

let's check how retry is handled in AWS client class,

```cpp
HttpResponseOutcome AWSClient::AttemptExhaustively(const Aws::Http::URI& uri,
    const Aws::AmazonWebServiceRequest& request,
    HttpMethod method,
    const char* signerName,
    const char* signerRegionOverride) const
{
    std::shared_ptr<HttpRequest> httpRequest(CreateHttpRequest(uri, method, request.GetResponseStreamFactory()));
    HttpResponseOutcome outcome;
    ...

    for (long retries = 0;; retries++)
    {
        outcome = AttemptOneRequest(httpRequest, request, signerName, signerRegionOverride);
        if (outcome.IsSuccess())
        {
            // blah
            break;
        }
        ...

        if (!m_retryStrategy->ShouldRetry(outcome.GetError(), retries))
        {
            break;
        }

        /// preparetions for retry

```

`S3Client` -> `AWSXMLClient` -> `AWSClient`. This is classical polymorphism and child classes can reuse above retry skeleton. If you check the stack trace above and ignore all parent classes, you will find at frame `#1`, the S3 specific implementation kicks in. Before we dig into S3 error handler, let's first look at `BuildAWSError`.

```cpp
AWSError<CoreErrors> AWSXMLClient::BuildAWSError(const std::shared_ptr<Http::HttpResponse>& httpResponse) const
{
    AWSError<CoreErrors> error;
    if (httpResponse->HasClientError())
    {
        bool retryable = httpResponse->GetClientErrorType() == CoreErrors::NETWORK_CONNECTION ? true : false;
        error = AWSError<CoreErrors>(httpResponse->GetClientErrorType(), "", httpResponse->GetClientErrorMessage(), retryable);
    }
    else if (httpResponse->GetResponseBody().tellp() < 1)
    {
        auto responseCode = httpResponse->GetResponseCode();
        auto errorCode = GuessBodylessErrorType(responseCode);
        Aws::StringStream ss;
        ss << "No response body.";
        error = AWSError<CoreErrors>(errorCode, "", ss.str(), IsRetryableHttpResponseCode(responseCode));
    }
    else
    {
        ...
        error = GetErrorMarshaller()->Marshall(*httpResponse);
    }
```

There are 3 situations to handle:

* First, client-side network failures.

In AWS implementation, there are 2 types of client-side errors, `USER_CANCELLED` and `NETWORK_CONNECTION`, which is reported by underlying CURL client. `USER_CANCELLED` is unable to retry.

```cpp
    /// in CurlHttpClient.cpp
    if (curlResponseCode != CURLE_OK && shouldContinueRequest)
        {
            response->SetClientErrorType(CoreErrors::NETWORK_CONNECTION);
```

* Second, decide from HTTP response code if HTTP response body is empty.

Check [IsRetryableHttpResponseCode][1] implementation for details.

* Last, decide from exception names returned from server-side.

From the code above, `GetErrorMarshaller` returns an `AWSErrorMarshaller`, which reads the HTTP response body and parse the XML payload to get exception name and message. Well, why exception name and message instead of error code? I guess the AWS server-side Java code uses exception style, whereas this Cpp client uses no-exception (error handling) style.

```cpp
AWSError<CoreErrors> AWSErrorMarshaller::Marshall(const Aws::String& exceptionName, const Aws::String& message) const
{
    if(exceptionName.empty())
    {
        return AWSError<CoreErrors>(CoreErrors::UNKNOWN, "", message, false);
    }

    // process exception name
    Aws::String formalExceptionName;

    AWSError<CoreErrors> error = FindErrorByName(formalExceptionName.c_str());
    if (error.GetErrorType() != CoreErrors::UNKNOWN)
    {
        ...
        error.SetExceptionName(formalExceptionName);
        error.SetMessage(message);
        return error;
    }
    ...
}
```

Finally, S3 implementations kicks in. As you already guessed, `S3ErrorMarshaller` -> `XMLErrorMarshaller` -> `AWSErrorMarshaller`, polymorphism again.

```cpp
AWSError<CoreErrors> S3ErrorMarshaller::FindErrorByName(const char* errorName) const
{
  AWSError<CoreErrors> error = S3ErrorMapper::GetErrorForName(errorName);
  if(error.GetErrorType() != CoreErrors::UNKNOWN)
  {
    return error;
  }

  return AWSErrorMarshaller::FindErrorByName(errorName);
}
```

Let's look at what child class `S3ErrorMarshaller` does. It only overrides function `FindErrorByName`, which injects S3 marshalling strategy inside `XMLErrorMarshaller`, which **is a** `AWSErrorMarshaller`

```cpp
static const int NO_SUCH_UPLOAD_HASH = HashingUtils::HashString("NoSuchUpload");
...

AWSError<CoreErrors> GetErrorForName(const char* errorName)
{
  int hashCode = HashingUtils::HashString(errorName);

  if (hashCode == NO_SUCH_UPLOAD_HASH)
  {
    return AWSError<CoreErrors>(static_cast<CoreErrors>(S3Errors::NO_SUCH_UPLOAD), false);
  }
  ...
  return AWSError<CoreErrors>(CoreErrors::UNKNOWN, false)
```

The above code tries to generate errors (AwsError) from requests based on error name (or exception name). The __exception name__ is `429` (corresponding to HTTP code, too many requests), which indeed is not a valid error name by AWS standards. The valid name should be `SlowDownException` or `SlowDown`. That's why it recognizes it as `CoreErrors::UNKNOWN`. Since S3 error marshaller cannot recognize the error name, it will use base [AWSErrorMarshaller::FindErrorByName(errorName)](https://github.com/aws/aws-sdk-cpp/blob/2a1d72685720b9663df2bfe38d3e69c4cc87c417/aws-cpp-sdk-core/source/client/CoreErrors.cpp#L37) to find its corresponding error type, which results in `UNKNOWN` again.

The final error will be returned as below, wrapped with exception name (`429`) and exception message (`Application request rate limit exceeded`).

```cpp
AWSError<CoreErrors> AWSErrorMarshaller::Marshall(const Aws::String& exceptionName, const Aws::String& message) const {
    ...
    return AWSError<CoreErrors>(CoreErrors::UNKNOWN, exceptionName, "Unable to parse ExceptionName: " + exceptionName + " Message: " + message, false);
}
```

Let's [summarize](#summary).

[0]: https://github.com/aws/aws-sdk-cpp/blob/2a1d72685720b9663df2bfe38d3e69c4cc87c417/aws-cpp-sdk-core/source/client/CoreErrors.cpp#L121
[1]: https://github.com/aws/aws-sdk-cpp/blob/2a1d72685720b9663df2bfe38d3e69c4cc87c417/aws-cpp-sdk-core/include/aws/core/http/HttpResponse.h#L118
[2]: https://github.com/aws/aws-sdk-cpp/blob/master/aws-cpp-sdk-s3/source/S3Errors.cpp
