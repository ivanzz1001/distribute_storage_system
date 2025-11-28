# BucketPolicy与BucketACL


## 1. BucketPolicy与BucketACL的区别
在S3兼容的对象存储中，Bucket Policy(存储桶策略）和Bucket ACL(存储桶访问控制列表)都是用于控制存储桶和对象访问权限的机制，但它们之间存在一些关键区别：

### 1.1 粒度

- **Bucket ACL**：提供基本的读写权限控制，可以针对存储桶和对象设置权限。权限包括READ、WRITE、READ_ACP（读取ACL）、WRITE_ACP（写入ACL）和FULL_CONTROL。可以授权给预定义的组（如所有用户、认证用户等）或特定用户（通过AWS账户ID）。

- **Bucket Policy**：提供更细粒度的访问控制。使用JSON策略语言，可以指定允许或拒绝特定操作（如s3:GetObject、s3:PutObject等）的条件，包括基于IP地址、时间、引用来源等条件。

### 1.2 灵活性

- Bucket ACL：相对简单，只能设置一些基本的权限，无法基于条件进行精细控制。

- Bucket Policy：非常灵活，可以编写复杂的规则，例如允许特定IP范围的用户访问，要求使用HTTPS，或者要求多因素认证等。

### 1.3 适用场景

- Bucket ACL：适用于简单的权限需求，例如将存储桶设置为公开读取，或者授予另一个AWS账户对存储桶的访问权限。

- Bucket Policy：适用于复杂的权限需求，例如需要精细控制不同操作和条件的场景。

### 1.4 权限优先级

- 当同时使用Bucket Policy和Bucket ACL时，Bucket Policy的权限设置会覆盖Bucket ACL。具体来说，如果Bucket Policy显示拒绝，则无论Bucket ACL如何允许，访问都会被拒绝。反之，如果Bucket Policy允许，但Bucket ACL没有允许，则访问也会被拒绝。因此，通常建议使用Bucket Policy进行更精确的控制。

>ps: Auth环节，一般先进行signature校验，然后是BucketPolicy的检查，之后是BucketACL的检查


### 1.5 支持的操作

Bucket ACL：主要针对存储桶和对象的读写权限以及ACL本身的读写权限。

Bucket Policy：可以控制几乎所有的S3操作，包括存储桶和对象的操作，以及存储桶策略的操作等。

### 1.6 授权对象

Bucket ACL：可以授权给预定义的组（如AllUsers、AuthenticatedUsers）或特定用户（通过AWS账户ID）。

Bucket Policy：可以授权给用户、账户、服务、IP地址等，还可以使用IAM角色和联合身份等

## 2. BucketPolicy示例

- **示例1**
  ```json
  {
    "Statement": [
       {
        "Effect": "Allow",
        "Principal":{"AWS":["arn:aws:iam:::user/lzy-test-02_userId"]}, 
        "Action": ["s3:RestoreObject"],
        "Resource": ["arn:aws:s3:::lzy-bucket-glacier/*"]
       }
   ]
  }
  ```

- **示例2**

 ```json
  {
    "Statement": [
       {
        "Effect": "Allow",
        "Principal":"*", 
        "Action": ["s3:RestoreObject"],
        "Resource": ["arn:aws:s3:::lzy-bucket-glacier/*"]
       }
   ]
  }
  ```
