# 存储相关文章收集

- [base64转16进制工具](https://tool.hiofd.com/base64-convert-hex-online/)

- [wrk压测工具](https://github.com/wg/wrk/tree/master)

- [rclone](https://rclone.org/downloads/)

- [rclone github](https://github.com/rclone/rclone/tree/master/docs)

- [gorilla mux](https://github.com/gorilla/mux)

- [golang限流相关](golang.org/x/time/rate)

- [web framework benchmark](https://web-frameworks-benchmark.netlify.app/result?asc=0&l=go&metric=totalRequestsPerS&order_by=level512)

- [cosbench](https://github.com/intel-cloud/cosbench)

- [zap log](go.uber.org/zap)

- [urfave client](github.com/urfave/cli)

- [viper配置文件解析](github.com/spf13/viper)

- [画火焰图](https://github.com/brendangregg/FlameGraph)

- [foundationdb exporter](https://github.com/aikoven/foundationdb-exporter)

- [golang lru](github.com/hashicorp/golang-lru)

- [prometheus golang client](github.com/prometheus/client_golang/prometheus)

- [s3tester工具](https://github.com/s3tester/s3tester)

- [如何对minio进行性能测试和分析](https://cloud.tencent.com/developer/article/2235817)

- [minio warp](https://github.com/minio/warp)

- [crushstore参考](https://github.com/andrewchambers/crushstore.git)



# pathEscape

```go
// urlPathEscape escapes URL path the in string using URL escaping rules
//
// This mimics url.PathEscape which only available from go 1.8
func urlPathEscape(in string) string {
	var u url.URL
	u.Path = in
	return u.String()
}

// pathEscape escapes s as for a URL path.  It uses rest.URLPathEscape
// but also escapes '+' for S3 and Digital Ocean spaces compatibility
func pathEscape(s string) string {
	return strings.ReplaceAll(urlPathEscape(s), "+", "%2B")
}
```
