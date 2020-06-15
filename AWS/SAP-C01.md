### SAP-C01



#### [1. Amazon RDS High Availability](https://amazonaws-china.com/cn/rds/ha/?nc1=h_ls)

1. For your [MySQL](https://amazonaws-china.com/rds/mysql/), [MariaDB](https://amazonaws-china.com/rds/mariadb/), [PostgreSQL](https://amazonaws-china.com/rds/postgresql/), [Oracle](https://amazonaws-china.com/rds/oracle/), and [SQL Server](https://amazonaws-china.com/rds/sqlserver/) database (DB) instances, you can use Amazon RDS Multi-AZ deployments. 

2. The Amazon Aurora PostgreSQL and Amazon Aurora MySQL engines include additional High Availability options.Even with a single database instance, Amazon Aurora increases availability by replicating your data six ways across three Availability Zones. 

3. In addition, you can choose to run one or more Replicas in an Amazon Aurora DB cluster. If the primary instance in the DB cluster fails, RDS automatically promotes an existing Aurora Replica to be the new primary instance and updates the server endpoint so that your application can continue operation with no manual intervention. If no Replicas have been provisioned, RDS will automatically create a new replacement DB instance for you when a failure is detected.

#### [2. 计划的预留实例](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2-scheduled-instances.html)

1. 利用计划的预留实例 (计划实例)，可以以一年为期限购买具有指定的开始时间和持续时间，并且每日、每周或每月重复一次的容量预留。您应提前预留容量，以确定其在需要时可用。
2. 对于不持续运行，而是按固定的计划运行的工作负载，计划实例是一个很好的选择。



#### [3. 如何保护 Amazon S3 存储桶中的文件？](https://amazonaws-china.com/cn/premiumsupport/knowledge-center/secure-s3-resources/)

1. 限制对 S3 资源的访问权限。默认情况下，所有 S3 存储桶都是私有的，只有获得显式授权的用户才能访问。

2. 通过以下方式限制对您的 S3 存储桶或对象的访问：

   > - 编写 [AWS Identity and Access Management (IAM) 用户策略](https://docs.aws.amazon.com/AmazonS3/latest/dev/example-policies-s3.html)，以指定可以访问特定存储桶和对象的用户。IAM 策略提供了一种编程方法来管理多个用户的 Amazon S3 权限。
   >
   > - 编写[存储桶策略](https://docs.aws.amazon.com/AmazonS3/latest/dev/example-bucket-policies.html)，定义对特定存储桶和对象的访问权限。您可以使用存储桶策略来授予跨多个 AWS 账户的访问权限、授予公开或匿名权限，并根据条件允许或或阻止访问。
   > - 使用 [Amazon S3 阻止公有访问](https://docs.aws.amazon.com/AmazonS3/latest/dev/access-control-block-public-access.html)作为一种限制公开访问的集中化方法。阻止公有访问设置会覆盖存储桶策略和对象权限。
   > - 在您的存储桶和对象上设置[访问控制列表 (ACL)](https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html)。
   > - 启用 [MFA Delete](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingMFADelete.html)，这要求用户使用 Multi-Factor Authentication (MFA) 设备进行身份验证，然后才能删除对象或禁用存储桶版本控制。
   > - 设置[受 MFA 保护的 API 访问](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_configure-api-require.html)，这要求用户使用 AWS MFA 设备进行身份验证，之后才能调用某些 Amazon S3 API 操作。
   > - 如果您临时与另一个用户共享 S3 对象，请创建预签名 URL 以授予对该对象的限时访问权限。有关更多信息，请参阅[与其他用户共享对象](https://docs.aws.amazon.com/AmazonS3/latest/dev/ShareObjectPreSignedURL.html)。



#### [4. New – VPC Endpoint for Amazon S3](https://aws.amazon.com/jp/blogs/aws/new-vpc-endpoint-for-amazon-s3/)

1. **VPC Endpoint** for S3 are easy to configure, highly reliable, and provide a secure connection to S3 that does not require a gateway or NAT instances.

