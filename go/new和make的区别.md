# Go中new和make的区别

## new

官方文档中的解释为:```The new built-in function allocates memory. The first argument is a type,not a value, and the value returned is a pointer to a newly allocated zero value of that type.```简单理解就是```先分配内存空间,然后初始化为T类型的零值,返回的是是地址```.

```Go
    //使用new方式
    var num2 = new(int)
    fmt.Println(num2)  //0xc00001a0b8
    fmt.Println(*num2) //0
    //下面代码等同于上面
    var num *int
	i := 0
	num = &i
	fmt.Println(*num) //0
```

上面代码中，使用```new(int)```方式声明了变量```num2```,从打印结果可以看出,它返回的是一个指针地址,同时初始化的零值为0.

## make
在Go中，make只能用于```slice,map,channel```这三种,它的第一个参数同new一样也是类型,不是值.与new不一样的是,```make返回的是类型和它参数的类型一致,并且是初始化之后的```.new返回的是它参数类型的指针类型.

```Go
    ints := make([]int,5)
	fmt.Println(len(ints))
	for index, value := range ints {
		fmt.Printf("index is [%d],value is [%d]\n",index,value)
	}
```

## 总结
new和make最大的两点不同在于:

- 可创建的对象不同,make只能创建slice,map,channel.而new可以创建除make以外的其他类型.
- new返回的不是T类型,返回的是T类型的指针.而make返回的是T类型

