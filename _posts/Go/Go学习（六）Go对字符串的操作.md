---
title: "Go学习（六）Go对字符串的操作（转）"
date: 2017-01-10 15:48:28
tags: [Go]
categories: [Go]
---

之前项目中有个功能是获取当前月份第一天的日期的需求，也就是如果今天的日期是"20170110"，我需要获取到的日期是"20170101"。

- 方式一：通过time获取当月的第一天

```
//昨天的日期
yesterdayTime := time.Now().AddDate(0, 0, -1)
//上周的日期
lastweekTime := time.Now().AddDate(0, 0, -8)
```
缺点：无法确定今天与当月第一天间隔多少天，所以需要结合方式二一起使用

- 方式二：通过time分别获取当前日期的年，月，日，然后重新拼接

```
year := time.Now().Year()
month := time.Now().Month()
day := time.Now().Day()
```
缺点：month获取出来是英文January，而不是01

- 方式三：将日期转换成字符串，然后截取字符串，删除字符串的最后两位字符加上"01"，或者是直接替换最后两位字符

缺点：在Go官方strings包中并没有找到合适的截取或者替换方法，下面是转自别人总结的strings包的方法


```
# 获取当前时间戳
fmt.Println(time.Now().Unix())
# 1389058332

# 格式化当前时间
fmt.Println(time.Now().Format("2006-01-02 15:04:05"))
# 2014-01-07 09:42:20

# 时间戳转str格式化时间
str_time := time.Unix(1389058332, 0).Format("2006-01-02 15:04:05")
fmt.Println(str_time)
# 2014-01-07 09:32:12

# str格式化时间转时间戳
the_time := time.Date(2014, 1, 7, 5, 50, 4, 0, time.Local)
unix_time := the_time.Unix()
fmt.Println(unix_time)
# 389045004

# 使用time.Parse将str格式化时间转时间戳
the_time, err := time.Parse("2006-01-02 15:04:05", "2014-01-08 09:04:41")
if err == nil {
    unix_time := the_time.Unix()
    fmt.Println(unix_time)		
}
# 1389171881
```

- http://www.cnblogs.com/golove/p/3236300.html
- https://my.oschina.net/1123581321/blog/190942

### strings包方法

```
// 转换

func ToUpper(s string) string
func ToLower(s string) string
func ToTitle(s string) string

func ToUpperSpecial(_case unicode.SpecialCase, s string) string
func ToLowerSpecial(_case unicode.SpecialCase, s string) string
func ToTitleSpecial(_case unicode.SpecialCase, s string) string

func Title(s string) string

------------------------------

// 比较
func Compare(a, b string) int
func EqualFold(s, t string) bool

------------------------------

// 清理

func Trim(s string, cutset string) string
func TrimLeft(s string, cutset string) string
func TrimRight(s string, cutset string) string

func TrimFunc(s string, f func(rune) bool) string
func TrimLeftFunc(s string, f func(rune) bool) string
func TrimRightFunc(s string, f func(rune) bool) string

func TrimSpace(s string) string

func TrimPrefix(s, prefix string) string
func TrimSuffix(s, suffix string) string

------------------------------

// 拆合

func Split(s, sep string) []string
func SplitN(s, sep string, n int) []string

func SplitAfter(s, sep string) []string
func SplitAfterN(s, sep string, n int) []string

func Fields(s string) []string
func FieldsFunc(s string, f func(rune) bool) []string

func Join(a []string, sep string) string

func Repeat(s string, count int) string

------------------------------

// 子串

func HasPrefix(s, prefix string) bool
func HasSuffix(s, suffix string) bool

func Contains(s, substr string) bool
func ContainsRune(s string, r rune) bool
func ContainsAny(s, chars string) bool

func Index(s, sep string) int
func IndexByte(s string, c byte) int
func IndexRune(s string, r rune) int
func IndexAny(s, chars string) int
func IndexFunc(s string, f func(rune) bool) int

func LastIndex(s, sep string) int
func LastIndexByte(s string, c byte) int
func LastIndexAny(s, chars string) int
func LastIndexFunc(s string, f func(rune) bool) int

func Count(s, sep string) int

------------------------------

// 替换

func Replace(s, old, new string, n int) string

func Map(mapping func(rune) rune, s string) string

------------------------------------------------------------

type Reader struct { ... }

func NewReader(s string) *Reader

func (r *Reader) Read(b []byte) (n int, err error)
func (r *Reader) ReadAt(b []byte, off int64) (n int, err error)
func (r *Reader) WriteTo(w io.Writer) (n int64, err error)
func (r *Reader) Seek(offset int64, whence int) (int64, error)

func (r *Reader) ReadByte() (byte, error)
func (r *Reader) UnreadByte() error

func (r *Reader) ReadRune() (ch rune, size int, err error)
func (r *Reader) UnreadRune() error

func (r *Reader) Len() int
func (r *Reader) Size() int64
func (r *Reader) Reset(s string)

------------------------------------------------------------

type Replacer struct { ... }

// 创建一个替换规则，参数为“查找内容”和“替换内容”的交替形式。
// 替换操作会依次将第 1 个字符串替换为第 2 个字符串，将第 3 个字符串
// 替换为第 4 个字符串，以此类推。
// 替换规则可以同时被多个例程使用。
func NewReplacer(oldnew ...string) *Replacer

// 使用替换规则对 s 进行替换并返回结果。
func (r *Replacer) Replace(s string) string

// 使用替换规则对 s 进行替换并将结果写入 w。
// 返回写入的字节数和遇到的错误。
func (r *Replacer) WriteString(w io.Writer, s string) (n int, err error)
```

上述的方式一和方式三都能实现此需求，但是方式三因为在strings包中没找到按照字符串索引位置截取的方法，所以在网上找到了一个自己实现的SubString截取字符串的方法。下面是具体的方法实现，更多字符串用法可以查看参考文章中的内容。

### 字符串的截取

```
//截取字符串 start 起点下标 length 需要截取的长度
func Substr(str string, start int, length int) string {
	rs := []rune(str)
	rl := len(rs)
	end := 0

	if start < 0 {
		start = rl - 1 + start
	}
	end = start + length

	if start > end {
		start, end = end, start
	}

	if start < 0 {
		start = 0
	}
	if start > rl {
		start = rl
	}
	if end < 0 {
		end = 0
	}
	if end > rl {
		end = rl
	}

	return string(rs[start:end])
}

//截取字符串 start 起点下标 end 终点下标(不包括)
func Substr2(str string, start int, end int) string {
	rs := []rune(str)
	length := len(rs)

	if start < 0 || start > length {
		panic("start is wrong")
	}

	if end < 0 || end > length {
		panic("end is wrong")
	}

	return string(rs[start:end])
}
```

参考文章：

- http://studygolang.com/articles/4287
- http://www.cnblogs.com/golove/p/3236300.html
- http://blog.csdn.net/chenbaoke/article/details/40318423
- https://my.oschina.net/manville/blog/313216
- http://studygolang.com/articles/5085