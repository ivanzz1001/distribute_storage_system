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

- [运维观止：监控相关](https://wiki.eryajf.net/pages/2477.html#_3-%E5%AF%BC%E5%85%A5-dashboard-%E6%A8%A1%E6%9D%BF)

- [grafana documentation](https://grafana.com/docs/grafana/latest/)

- [仪表盘市场](https://grafana.com/grafana/dashboards/)

- [Grafana 中文入门教程 | 构建你的第一个仪表盘](https://cloud.tencent.com/developer/article/1807679)

- [模板化Dashboard](https://doc.cncf.vip/prometheus-handbook/part-ii-prometheus-jin-jie/grafana/templating)
