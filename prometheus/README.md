# 相关文章收集

- [prometheus metrics type](https://prometheus.io/docs/concepts/metric_types/)

- [prometheus querying](https://prometheus.io/docs/prometheus/latest/querying/basics/)

  ```text
  示例:
  sum by(api)(rate(http_request_duration_count{api=~"$api", job=~"$job",service="$service"}[1m]))
  ```

- [通过Prometheus+grafana搭建可视化监控](https://tech.qimao.com/ce-shi-2/)

- [grafana datasources](https://grafana.com/docs/grafana/latest/datasources/prometheus/)

- [prometheus golang client](github.com/prometheus/client_golang/prometheus)

- [prometheus golang_client document](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus)
