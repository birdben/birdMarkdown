
如何查看logstash插件的版本

进入logstash安装目录，然后搜索对应安装插件的目录

```
$ cd ${LOGSTASH_HOME}
$ ls -la vendor/bundle/jruby/1.9/gems | grep "logstash-filter-split"
drwxrwxr-x   3 elk elk  4096  7月  7 08:00 logstash-filter-split-2.0.5
```

参考文章：
- http://stackoverflow.com/questions/39114563/get-plugin-version-installed-in-logstash