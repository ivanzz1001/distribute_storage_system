# 模拟客户端不读取http响应数据流直接断开连接的情况

```python
import boto3

s3 = boto3.client('s3',
                  aws_access_key_id='aaaaa',
                  aws_secret_access_key='bbbb',
                  endpoint_url='http://127.0.0.1:80'  # 如果使用非AWS S3
                  )

bucket_name = 'test-bucket'
object_key = 'video/8_0001.mp4'
start_byte = 0
end_byte = 2815224  # 包含在内

response = s3.get_object(
    Bucket=bucket_name,
    Key=object_key,
    Range=f'bytes={start_byte}-{end_byte}'
)

# 读取返回的数据流
#partial_data = response['Body'].read()

# 将数据写入文件
#with open('local_partial_file.bin', 'wb') as f:
#    f.write(partial_data)
```
