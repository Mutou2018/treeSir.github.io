---
title: GoLang学习01
date: 2021-03-28 18:52:36
tags:
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/%E7%8B%97%E7%8B%97.jpg
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/golang.png
---
# GoLang学习01

*作为一个有志向的前端怎么能不学习下后端知识呢？不能老是被后端忽悠😂*

​		作者之前已经学过一段时间的golang了,但是还是需要记录一下,作为自己的学习记录也不错。

- 为什么学习go

  golang作为一款明星级别的语言，金主爸爸是Google，稳定性也比较强，而且golang编译后直接生成的是机器可阅读的语言，和Java不一样，可直接运行。golang和java是两个不一样的方向。况且golang已经出来很久了，作为我们项目边缘组物联网相关，所以golang也是我们项目的主力开发语言。话不多说，直接🐛🐛🐛。

- start
  go的环境搭建网上已经有很多了，不多赘述。作为程序员的一句经典名言，don`t talk too much,show me the code。
  第一个程序肯定是经典的Hello world！

  新建一个文件 hello.go

  ```go
  package main
  
  import "fmt"
  
  func main(){
      fmt.Println("hello world")
  }
  
  ```

  直接在命令行输入：go run hello.go

  展示出hello world，表明程序已经成功运行了。

- 变量和基础类型

  变量就是存储特定类型的值而提供给内存位置的名称。go中声明变量的方法有多种。

  ```go
  //第一种
  var name type 
  name = value
  //第二种
  var name = value
  //第三种
  name := value
  
  var a int = 100
  var b = 100
  c := 100
  ```

  go也是需要通过静态语言的，不是动态类似js（很多时候js都在为这个填坑），这样在开发过程中可以避免大半的错误。

  go有个好处就是类型推导，``` var a = 10```，可以不明确指定类型，给喜欢偷懒的我们提供了方便。还有一个更便捷的方法就是 ```c := 100 ```。作为懒狗一条的我，肯定是喜欢这种方式。

  go语言的基本类型有：

  - bool
  - string
  - int，int8，int16，int32，int64
  - uint、uint8、uint16、uint32、uint64、uint
  - byte // uint8 的别名
  - rune // int32 的别名 代表一个 Unicode 码
  - float32 float64
  - complex64、complex128

  字符串：

  ​	当需要使用到多行字符串时，可以使用反引号``` ` ```

  ```
  const str = `第一行
  第二行
  ...
  `
  ```

  打印函数：

  ​	fmt.Print()

  格式化打印常用的占位符：

  ​	

  |              |              |
  | ------------ | ------------ |
  | %v:原样输出  | %T：打印类型 |
  | %s:bool类型  | %s：字符串   |
  | %c：打印字符 | %p：打印地址 |

  流程语句：

  ​		流程语句就是语言都有的流程控制，if，switch，for循环等，其中goto是一个相对于js很有意思的东西

  - goto：

    可以无条件的转移到过程中指定的行。

    ```
        if err != nil {
            goto onExit
        }
        err = secondCheckError()
        if err != nil {
            goto onExit
        }
        fmt.Println("done")
        return
    onExit:
        fmt.Println(err)
        exitProcess()
    ```

    统一处理错误这种类似的情景。

    

