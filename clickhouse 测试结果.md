# clickhouse 测试结果


## 测试环境
### 硬件
* 处理器: 2.7 GHz 双核Intel Core i5
* 内存: 16 GB 1867 MHz DDR3

### 软件环境
* NodeJs 15.10.0
* NPM 7.5.3
* Clickhouse 22.7.1
* Redis 7.0.2 (Docker)
* WiFi 网络连接方式

---
## 数据样本
* 模拟数据量 133537, 多次使用

---
## 测试方法
* 写入测试
    1. A 机器发送 udp 发送二进制样本数据, 请求间隔 5ms
    2. B 机器接收 udp 数据
    3. 解析二进制数据为 Json
    4. 将 Json 缓存到 Redis 链表中
    5. B 机器定时将 Redis 链表中的数据批量写入 Clickhouse. 频率为 1000 rows / s
* 只读测试
    1. 编写 4 个不停的 SQL 随机执行, 每个 SQL 基本覆盖了 800W 的数据.
    2. 定时任务每秒执行一次 SQL 查询
    3. SQL 列表
    ``` sql
        const sqlArray = [
            // 查询总共有几个飞艇
            `
            SELECT
                airId,
                count(*) AS num
            FROM ft.test01
            GROUP BY airId
            ORDER BY num DESC
            `,
            // 查询时间段内有几个飞艇
            `
            SELECT
                airId,
                count(*) AS num
            FROM ft.test01
            WHERE toYYYYMMDD(time) >='20220708'
            GROUP BY airId
            ORDER BY num DESC
            `,
            // 查询每个飞艇在哪些天有记录
            `
            SELECT
                airId,
                toYYYYMMDD(time) as tt,
                count(*) AS num
            FROM ft.test01
            GROUP BY airId, tt
            ORDER BY airId, tt asc
            `,
            // 每个飞艇的最后电量数据
            `
            SELECT
                airId,
                battery_SOC,
                last_time
            FROM (
                SELECT
                    airId,
                    battery_SOC,
                    time,
                    first_value(time) OVER w AS last_time,
                    row_number() OVER w AS row_num
                FROM ft.test01
                WINDOW w AS (partition by airId ORDER BY time DESC)
            )
            WHERE row_num = 1
            `,
        ]

    ```
* 读写测试
    1. "写入测试" + "只读测试" 同时

---
## 测试数据
### 写库测试
| 当前表总量 | 进程数 | 写库总量 | 库差值 | 请求量 | 耗时 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 593,716 |  1 | 133,378 | 159 | 133,378 | 11:32 |
| 727,094 | 1 | 133,446 | 91 | 133,446 | 11:33 |
| 1,461,229 | 5 | 552,880 |  | 552,880 | 11:41 |
| 2,014,109 | 5 | 552,957 |  | 552,957 | 11:22 |
| 4,189,977 | 5 | 552,406 |  | 552,406 | 11:34 |
| 4,742,383 | 5 | 552,358 |  | 552,358 | 11:34 |

说明:
* 进程数 5 的"库差值"为空是因为有一个进程出现了错误.
---


### 只读测试
![image](https://github.com/redrum0003/redrum0003/raw/main/only_read.png)

### 读写测试
![image](https://github.com/redrum0003/redrum0003/blob/main/read_write.png)


---
## 测试结果

1. 在没有缓存请情况下 1 个进程, 小于 20 ms Clickhouse 就会报错.
2. 加入缓存和批量写入策略后, 即使增加到 5 个进程耗时没有明显的变化.
3. Redis 因为只做临时存储少量数据, 在 5 进程时内存消耗 4 ~ 10 MB
4. 只读测试平均在 1.8s 左右, 最高 2.68s.
5. 读写测试平均在 8.17s 左右, 最高 15.2s.
6. 读写测试比只读测试耗时高 4 ~ 7 倍 (剔除读写测试尾部的只读测试结果)


---

## 问题
1. udp 有丢包现象, 从上表的"库差值"可以看出.
2. 部署风险: 目前clickhouse 没有基于 Windows 的部署包, 只能通过虚拟机来实现安装. 如果目标机器无法提供运行虚拟机环境, 那么部署将无法完成.
3. Windows 环境部署性能消耗: 在可提供虚拟机环境的前提下, 在虚拟机中运行数据库, 虚拟机本身的性能消耗, 稳定性方面有担心.


