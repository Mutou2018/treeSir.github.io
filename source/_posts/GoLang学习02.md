---
title: GoLang学习02
date: 2021-04-13 20:52:36
tags: Golang
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/%E7%8B%97%E7%8B%97.jpg
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/golang.png
---
# GoLang学习02

- 数组和切片

  数组是具有相同唯一类型的一组已编号且长度固定的数据项序列。数组一旦定义，大小不能更改。

  数组其中比较关键的就是：*唯一类型、长度固定、大小不可更改* 

  golang里面数组是值类型，不是引用类型！

  1.数组

  - 创建数组

  //声明数组
    var name [size] type

    ```go
    
    var arr [10]int
  var arr2 = [5]int{1,2,3,4,5}
    var arr3 = [3][4]{
      {1,2,3,4},
        {5,6,7,8},
      {9,10,11,12}
    }
    ```

    

  - 遍历数组

    ```go
    var n [10]int
    for i:=0;i<10;i++{
      n[i] = i+1
       	fmt.Printf("n[%d] = %v\n",i,n[i])
  }
    for idx,val := range n{
    		fmt.Printf("n[%v] = %v\n",idx,val)
    	}
    ```

    遍历数组可以使用for，也可以使用range函数。

  - 数组方法

    a := [...]int｛1，2，3，4｝

    len(a) //4

  2.切片

  ​	切片是对数组的抽象。go的数组长度不可变，但是平时日常工作中很不方便。切片就诞生了，动态数组，可以追加元素。

  ​	切片本身没有任何数据，它是对现有数组的引用。

  ​	切片包含了三个元素：

  ​	 1）指针

  ​	2）长度

  ​	3）最大长度	

  - 创建切片

    ```go
    var val []type
    var slilce1 []type = make([]type,len)
    slice2 := make([]type,len)
    //make([]T,len,cap)
    ```

  - 切片初始化

    ```go
    s[index] = val
    s := []int{1,2,3} //不指定长度
    s := arr[startIndex:endIndex]
    ```

  - 修改切片

    slice没有自己的任何数据.它只是底层的数组地址,对slice做的任何修改都会反映在之前的数组中。

  - 切片的函数

    - len（）和cap（）

      len（）获取长度

      cap（）获取容量

    - append（）和copy()

      ```go
      var number = []int
      number = append(number,1) //增加单个元素
      number = append(number,2345)//增加多个元素
      var number2 = make([]int,len(number),(cap(number)*2))
      copy(number2,number) //将number2拷贝到number中
      number发生变化时，number2不会改变，copy方法不会建立两者之间的联系
      ```

      

- Map（集合）

  map是无序的键值对的集合。map最重要的一点就是通过key来快速检索数据。
  
  map的几点特性：
  
  - map是无序的
  - map的长度不固定，是一种引用类型
  - 内置len（）同样适用于map
  - map的key可以是所有可比较的类型。
  
  ```go
  创建map
  	var m1 map[key_data_type]value_data_type
  	m1 = make(map[key_data_type]value_data_type)
  	rate := map[string]float {"c":1,"Go":4.5}
  ```
  
  1. 删除map-delete()函数
  
  ​		delete(map,key)函数用于删除集合的元素。删除函数不返回任何值。
  
  2. 获取map的值
  
     通过key获取map中对应的value值。
  
     但是key不存在时，会得到value值类型的默认值。可以通过ok-idiom获取值。
  
     x,ok := m["b"]
  
  3. map是引用类型