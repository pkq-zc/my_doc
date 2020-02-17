# ES-ik分词器安装

该安装地址可以参考github开源项目[elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)

## 手动安装

- 手动下载安装包，安装包地址：```https://github.com/medcl/elasticsearch-analysis-ik/releases```，需要注意的是要下载与自己版本一致的，版本不一致的可能会有问题。
- 在es的安装地址下，```plugins```文件夹中创建目录```ik```
- 解压安装包到```ik```文件夹中

### 使用es命令安装

- 进入es安装目录```bin```中
- 执行命令```/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.3.0/elasticsearch-analysis-ik-6.3.0.zip```，同样需要注意的是，将```6.3.0```替换成自己的版本。

### 测试使用

安装完成之后，重启es,启动过程打印的日志应该会有下面内容：
![启动日志](https://i.loli.net/2020/02/17/5EyVvljTGWxM3Yu.png)
可以看出，加载了我们刚刚安装好的```anlysis-ik```.我们可以使用```_analyzer```测试使用。

```
POST _analyze
{
  "text": "美国留给伊拉克的是一个烂摊子吗？",
  "analyzer": "ik_smart"
}

POST _analyze
{
  "text": "美国留给伊拉克的是一个烂摊子吗？",
  "analyzer": "ik_max_word"
}
```

运行上面的测试示例，将会得到分词的结果：

结果一： 

```
{
  "tokens" : [
    {
      "token" : "美国",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "留给",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "伊拉克",
      "start_offset" : 4,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "的",
      "start_offset" : 7,
      "end_offset" : 8,
      "type" : "CN_CHAR",
      "position" : 3
    },
    {
      "token" : "是",
      "start_offset" : 8,
      "end_offset" : 9,
      "type" : "CN_CHAR",
      "position" : 4
    },
    {
      "token" : "一个",
      "start_offset" : 9,
      "end_offset" : 11,
      "type" : "CN_WORD",
      "position" : 5
    },
    {
      "token" : "烂摊子",
      "start_offset" : 11,
      "end_offset" : 14,
      "type" : "CN_WORD",
      "position" : 6
    },
    {
      "token" : "吗",
      "start_offset" : 14,
      "end_offset" : 15,
      "type" : "CN_CHAR",
      "position" : 7
    }
  ]
}
```

结果二：

```
{
  "tokens" : [
    {
      "token" : "美国",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "留给",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "伊拉克",
      "start_offset" : 4,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "的",
      "start_offset" : 7,
      "end_offset" : 8,
      "type" : "CN_CHAR",
      "position" : 3
    },
    {
      "token" : "是",
      "start_offset" : 8,
      "end_offset" : 9,
      "type" : "CN_CHAR",
      "position" : 4
    },
    {
      "token" : "一个",
      "start_offset" : 9,
      "end_offset" : 11,
      "type" : "CN_WORD",
      "position" : 5
    },
    {
      "token" : "一",
      "start_offset" : 9,
      "end_offset" : 10,
      "type" : "TYPE_CNUM",
      "position" : 6
    },
    {
      "token" : "个",
      "start_offset" : 10,
      "end_offset" : 11,
      "type" : "COUNT",
      "position" : 7
    },
    {
      "token" : "烂摊子",
      "start_offset" : 11,
      "end_offset" : 14,
      "type" : "CN_WORD",
      "position" : 8
    },
    {
      "token" : "摊子",
      "start_offset" : 12,
      "end_offset" : 14,
      "type" : "CN_WORD",
      "position" : 9
    },
    {
      "token" : "吗",
      "start_offset" : 14,
      "end_offset" : 15,
      "type" : "CN_CHAR",
      "position" : 10
    }
  ]
}

```