# ES-pinyin分词器安装

该安装地址可以参考github开源项目[elasticsearch-analysis-pinyin](https://github.com/medcl/elasticsearch-analysis-pinyin)

## 手动安装

- 手动下载安装包，安装包地址：```https://github.com/medcl/elasticsearch-analysis-pinyin/releases```，需要注意的是要下载与自己版本一致的，版本不一致的可能会有问题。
- 在es的安装地址下，```plugins```文件夹中创建目录```pinyin```
- 解压安装包到```pinyin```文件夹中

### 使用es命令安装

- 进入es安装目录```bin```中
- 执行命令```/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v7.4.0/elasticsearch-analysis-pinyin-7.4.0.zip```，同样需要注意的是，将```7.4.0```替换成自己的版本。

### 测试使用

```
POST _analyze
{
  "analyzer": "pinyin",
  "text": "美国"
}
```

运行上面的测试示例，结果如下:

```
{
  "tokens" : [
    {
      "token" : "mei",
      "start_offset" : 0,
      "end_offset" : 0,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "mg",
      "start_offset" : 0,
      "end_offset" : 0,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "guo",
      "start_offset" : 0,
      "end_offset" : 0,
      "type" : "word",
      "position" : 1
    }
  ]
}
```

### 插件组成

这个插件包含:analyzer:```pinyin``` ,tokenizer:```pinyin```和token-filter:```pinyin```,同时支持部分自定义选项.

- ```keep_first_letter```:这个参数会将词的第一个字母全部拼起来.例如:刘德华->ldh.默认为:true
- ```keep_separate_first_letter```:这个会将第一个字母一个个分开.例如:刘德华->l,d,h.默认为:flase.如果开启,可能导致查询结果太过于模糊,准确率太低.
- ```limit_first_letter_length```:设置最大```keep_first_letter```结果的长度,默认为:16
- ```keep_full_pinyin```:如果打开,它将保存词的全拼,并按字分开保存.例如:刘德华> [liu,de,hua],默认为:true
- ```keep_joined_full_pinyin```:如果打开将保存词的全拼.例如:刘德华> [liudehua],默认为:false
- ```keep_none_chinese```:将非中文字母或数字保留在结果中.默认为:true
- ```keep_none_chinese_together```:保证非中文在一起.默认为: true, 例如: DJ音乐家 -> DJ,yin,yue,jia, 如果设置为:false, 例如: DJ音乐家 -> D,J,yin,yue,jia, 注意: ```keep_none_chinese```应该先开启.
- ```keep_none_chinese_in_first_letter```:将非中文字母保留在首字母中.例如: 刘德华AT2016->ldhat2016, 默认为:true
- ```keep_none_chinese_in_joined_full_pinyin```:将非中文字母保留为完整拼音. 例如: 刘德华2016->liudehua2016, 默认为: false
- ```none_chinese_pinyin_tokenize```:如果他们是拼音,切分非中文成单独的拼音项. 默认为:true,例如: liudehuaalibaba13zhuanghan -> liu,de,hua,a,li,ba,ba,13,zhuang,han, 注意: ```keep_none_chinese```和```keep_none_chinese_together```需要先开启.
- ```keep_original```:是否保持原词.默认为:false
- ```lowercase```:小写非中文字母.默认为:true
- ```trim_whitespace```:去掉空格.默认为:true
- ```remove_duplicated_term```:保存索引时删除重复的词语.例如: de的>de, 默认为: false, 注意:开启可能会影响位置相关的查询.
- ```ignore_pinyin_offset```:在6.0之后,严格限制偏移量,不允许使用重叠的标记.使用此参数时,忽略偏移量将允许使用重叠的标记.请注意,所有与位置相关的查询或突出显示都将变为错误,您应使用多个字段并为不同的字段指定不同的设置查询目的.如果需要偏移量,请将其设置为false。默认值:true

