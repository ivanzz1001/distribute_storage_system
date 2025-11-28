# BucketPolicy与BucketACL的区别

在S3兼容的对象存储中，Bucket Policy(存储桶策略）和Bucket ACL(存储桶访问控制列表)都是用于控制存储桶和对象访问权限的机制，但它们之间存在一些关键区别：

1. 粒度

  - Bucket ACL：提供基本的读写权限控制，可以针对存储桶和对象设置权限。权限包括READ、WRITE、READ_ACP（读取ACL）、WRITE_ACP（写入ACL）和FULL_CONTROL。可以授权给预定义的组（如所有用户、认证用户等）或特定用户（通过AWS账户ID）。

  - Bucket Policy：提供更细粒度的访问控制。使用JSON策略语言，可以指定允许或拒绝特定操作（如s3:GetObject、s3:PutObject等）的条件，包括基于IP地址、时间、引用来源等条件。

1. 

