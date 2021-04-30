---
layout: post
title: "AWS CPP SDK cannot resolve hostname"
subtitle: "virtual-hosted style vs path-style"
date: 2020-04-08 21:02:08
categories: [aws, cpp]
tag: [aws, cpp]
---

Recently, I upgraded the [AWS CPP SDK](https://github.com/apple/turicreate/pull/3029) from `0.40`, which was integrated to `turicreate` codebase around 2017, to `1.7.143`.

Later, I [rewrote our S3 path](https://github.com/apple/turicreate/pull/2920) with the latest SDK and deprecated the old CURL based implementation. After this refactoring, `turicreate` can interact with S3 that uses **SIGV4** authentication, which is the ultimate motivation. But the new code path has regression with our legacy S3 service. It cannot resolve the host name of the legacy S3 proxy. As shown in the log trace,

```shell
[ERROR] 2020-04-08 21:44:24.124 AWSClient [0x1079395c0] HTTP response code: -1
Exception name:
Error message: curlCode: 6, Couldn't resolve host name
```

The corresponding request is shown below.

```bash
DEBUG] 2020-04-08 21:44:24.119 AWSAuthV4Signer [0x1079395c0] Canonical Request String: GET
/
delimiter=%2F&list-type=2&prefix=sa
content-type:application/xml
host:gui-public.foo.com
```

The S3 url is `S3://foo.com/gui-public/some-key` and the host should `foo.com`, and I cannot ping `gui-public.foo.com` since it doesn't exist to my knowledge. Through a long time research, I realize this is [virtual hosted style][0].
[virtual host][0] is an endpoint to the bucket server directly. When a request to the key behind a bucket is made, it goes directly to that bucket server to ask data, instead of going to an aws endpoint and then letting the server behind that endpoint to redirect it to the bucket server.

Our legacy S3 proxy was built around 2017 and it only supports [path-style][0] requests. The CPP SDK is first introduced to `turicreate` in 2017, when [virtual hosted style][0] isn't available. It only supports and uses path-style.

In order to solve my problem here, I need to explicitly tell the SDK to use path-style.

```c++
Aws::S3::S3Client client(credentials, clientConfiguration,
                        /* default sigv4 policy value, for performance */
                        Aws::Client::AWSAuthV4Signer::PayloadSigningPolicy::Never,
                        /* use virtual host */ false);
```

However, I can still use [aws cli version 1](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) to access the keys from proxies that support either virtual host style or path style. I suspect `aws cli version 1` uses path style by default and it can fallback to virtual host style.

Probably CPP SDK should behave in the same way in order to be backwards compatible. Based on the [post][0], AWS wants to deprecate path style completely in order to reduce the traffics to the load balancers behind the few common S3 endpoints,

> anticipating a world with billions of buckets homed in many dozens of regions, routing all incoming requests directly to a small set of endpoints makes less and less sense over time. DNS resolution, scaling, security, and traffic management (including DDoS protection) are more challenging with this centralized model. The virtual-hosted model reduces the area of impact (which we call the “blast radius” internally) when problems arise; this helps us to increase availability and performance.

From this point of view, I can understand why they set virtual host style as default, which contradicts with backwards compatibility design pattern.

[0]: https://aws.amazon.com/blogs/aws/amazon-s3-path-deprecation-plan-the-rest-of-the-story
