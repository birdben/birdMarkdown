## ELK实战 —— 收集浏览器的浏览历史记录（一）

### 这里我们只收集Safari，Chrome，FireFox三个主流浏览器

这三个主流的浏览器都是使用的sqlite数据库来存储浏览历史记录的，所以我们要想收集sqlite的数据，需要先安装logstash-input-sqlite插件

插件的GitHub地址：

- https://github.com/logstash-plugins/logstash-input-sqlite

### sqlite插件的github上给出了两种安装方式

- 方式一：修改Gemfile文件方式安装（推荐使用）
- 方式二：自行使用gem命令build插件

#### 安装步骤

```
# 编辑Gemfile文件，在文件的最后面添加下面的配置
# path：可以下载插件到本地通过path指定本地路径安装，如果自己本地有修改sqlite插件需要指定本地插件的路径
gem "logstash-filter-awesome", :path => "/your/local/logstash-filter-awesome"
# 这里我们使用默认安装，不需要指定path
gem "logstash-input-sqlite"

# 安装插件
# Logstash 2.3以上的版本使用
bin/logstash-plugin install --no-verify
# Logstash 2.3之前的版本使用
bin/plugin install --no-verify
```

#### 无法安装插件

如果无法安装logstash的插件，很有可能是logstash插件数据源被墙了，所以Gemfile配置文件最好使用国内的数据源配置，一般都能安装成功的。

修改Gemfile文件的gem数据源地址

```
# source "https://rubygems.org"
source "https://ruby.taobao.org/"
```

#### 本地安装插件

有的插件在淘宝的gem库中找不到，这时候可以考虑本地安装的办法。

先去https://github.com/logstash-plugins下载对应的插件，然后解压，在logstash的Gemfile中添加一行(以logstash-output-webhdfs为例)：

```
# 编辑Gemfile文件，在文件的最后面添加下面的配置
gem "logstash-output-webhdfs", :path => "/home/ec2-user/logstash-output-webhdfs"

bin/plugin install --no-verify
```

参考文章：

- https://www.elastic.co/guide/en/logstash/current/plugins-inputs-sqlite.html
- http://www.jianshu.com/p/4fe495639a9a
- https://github.com/logstash-plugins/logstash-input-sqlite