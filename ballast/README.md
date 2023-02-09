# Test Setup
`make` runs the `generate` command of the operator and creates an individual manifest for each test configuration (`tempo-*.yaml`).
Each tempo cluster runs in its own namespace, including minio for s3 compatible storage and one or more `grafana/xk6-client-tracing` load generators.
The load generator is configured with 1000 virtual users creating traces every 1 to 5 seconds.
Based on the test configuration, there can be multiple replicas of the same load generator.

# Configurations
| Setting       | Values            |
| ------------- | ----------------- |
| Ballast       | 0, 1gb, 2gb       |
| GOMEMLIMIT    | default, 2gb, 4gb |
| Virtual Users | 0, 1000, 3000     |
| Tempo version | 1.5               |
| Go version    | 1.19.5            |

# Results and Conclusion
[See Grafana dashboards](grafana/).

All results are shown with a total of 3000 virtual users (1000 virtual users generating traces every 1-5 seconds **x** 3 load generator instances).

| Metric                                       | defaults<sup>1</sup> | 1 GiB ballast | 2 GiB ballast | 2 GiB memlimit | 4 GiB memlimit  |
| -------------------------------------------- | -------------------- | ------------- | ------------- | -------------- | --------------- |
| Distributor avg. heap size<sup>2</sup> [GiB] | 2.79                 | 3.41          | 3.77          | 1.80           | 2.75            |
| Ingester    avg. heap size<sup>2</sup> [GiB] | 1.47                 | 2.02          | 2.43          | 1.62           | 1.74            |
| Distributor GC collection rate [ops/min]     | 3.15                 | 1.95          | 1.59          | 80.7           | 3.17            |
| Ingester    GC collection rate [ops/min]     | 6.56                 | 2.85          | 1.66          | 19.1           | 5.81            |

<sup>1</sup> no ballast, default GOMEMLIMIT

<sup>2</sup> excluding ballast

The results show a difference in the Garbage Collector collection rate for tests with and without ballast.
Based on the absolute numbers, I think 1-5 additional GC operations *per minute* are negligible in practice in this test setup.
It's possible that there is a bigger impact of setting ballast in larger deployments.

The tests with `GOMEMLIMIT=2GiB` show a significant higher CPU usage and GC collection rate than the other test configurations.
For the distributor component, the Go runtime manages to keep the heap under 2 GiB at the expense of CPU overhead compared to the other configurations.
Towards the end of the measured timeframe, we can see the Go runtime excessively running GC collections for the ingester, trying (but failing) to keep the heap size under 2 GiB.
This results in excess CPU usage (heap thrashing).

The load generator didn't issue any queries, therefore the querier and query-frontend components didn't show any significant differences.

# Reading Material
* https://github.com/golang/go/issues/23044
* https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap/
* https://www.uber.com/en-US/blog/how-we-saved-70k-cores-across-30-mission-critical-services/
* https://go.dev/doc/gc-guide#Memory_limit
* https://weaviate.io/blog/gomemlimit-a-game-changer-for-high-memory-applications
* https://www.sobyte.net/post/2022-06/memory-ballast-gc-tuner/
