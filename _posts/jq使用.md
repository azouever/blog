---
title: jq使用
date: 2024-01-09 11:44:34
tags:
  - json
---
# 日常使用

```
// 提取数组中的元素
// 主要的使用场景是从curl的结果中解析一些数据,这里的result是一个数组

echo '{"result":[{"msg":"hello"},{"msg":"world"}]}' | jq '.result|.[]|.msg'

{
  "result": [
    {
      "msg": "hello"
    },
    {
      "msg": "world"
    }
  ]
}

curl *******
 --compressed | jq '.result|.[] |.msg' >> chat_timeout.xkx



// 数据过滤(简单数组)
echo '["apple", "banana", "cherry", "date"]' | jq '.[] | select(. | test("a$") | not)'

echo '["apple", "banana", "cherry", "date"]' | jq '.[] | select(. | test("a$") )'

// 数据过滤(从复杂对象数组中选择)
echo '[{"msg":"hello","code":"34"},{"msg":"world","code":"56"}]' | jq '.[] | select(.["code"] == "34") | .["msg"]' | tr -d '"'


```