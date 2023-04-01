---
title: CCFtab
date: 2020-08-28 15:11:22
tags:
	-CCF
---

## 目录

1.常用注意要点

2.常用stl详解以及使用方式简述

<!-- more -->

+  2.1 map
+ 2.2 unordered_map、multimap
+ 2.3 vector
+ 2.4 queue
+ 2.5 stack
+ 2.6 set

3.常用函数原型以及解释（随时添加）

​	`#includ<ctype.h>`

​	`#include<algorithm>`

​	`#include<string>`

4.运用文件读写加快测试速度的代码

---



### 常用注意要点

#### 1.#include<bits/stdc++.h>

万能头文件，包含了所有的已知常用库

#### 2.ios::sync_with_stdio(false);

为了防止因为cin以及cout产生超时，将cin以及cout的缓冲区置为false

#### 3.cin.tie(0);

接触cin与cout的绑定，加快执行效率

### 常用stl详解以及使用方式

stl中很常用的一个东西就是迭代器iterator，但是注意，iterator相当于是指向节点的指针，所以在整个stl发生改变或者删除时会出现迭代器产生空指针的情况，所以要牢记一点：不要用过期的iterator!!!

如果希望将一个结构体作为几种stl的键值时需要将struct中的某一个值作为键值对`<`进行重载，具体方法如下：

```c++
struct Info  
{  
    string name;  
    float score;  
    //重载“<”操作符，自定义排序规则  
    bool operator < (const Info &a) const  
    {  
        //按score从大到小排列  
        return a.score<score;  
    }  
}  
set<Info> s;  
......  
set<Info>::iterator it;  
```



#### 1.map

map是键-值对的组合，有以下的一些定义的方法：

- `map<k, v> m;`
- `map<k, v> m(m2);`
- `map<k, v> m(b, e);`

上述第一种方法定义了一个名为m的空的map对象；第二种方法创建了m2的副本m；第三种方法创建了map对象m，并且存储迭代器b和e范围内的所有元素的副本。

map的value_type是存储元素的键以及值的pair类型，键为const。

##### 插入

insert函数的插入方法主要有如下：

- `m.insert(e)`
- `m.insert(beg, end)`
- `m.insert(iter, e)`

上述的e一个value_type类型的值。beg和end标记的是迭代器的开始和结束。

两种插入方法如下面的例子所示：

```c++
#include <stdio.h>
#include <map>
using namespace std;

int main(){
        map<int, int> mp;
        for (int i = 0; i < 10; i ++){
                mp[i] = i;  //这种方法很特殊要记住，在插入后回覆盖键值
        }
        for (int i = 10; i < 20; i++){
                mp.insert(make_pair(i, i));
        }
        map<int, int>::iterator it;
        for (it = mp.begin(); it != mp.end(); it++){
                printf("%d-->%d\n", it->first, it->second);
        }
        return 0;
}
```

Map中可以利用结构体作为key值，但是要对小于号进行符号重载

例如：

```c++
struct Point
{
	ll x;
	ll y;
	bool operator < (const Point & a) const
	{
		return x<a.x;
	}
};
```

##### 查找

上述采用下标的方法读取map中元素时，若map中不存在该元素，则会在map容器中插入一个新的元素。

因此，若只是查找该元素是否存在，可以使用函数`count(k)`，该函数返回的是k出现的次数；若是想取得key对应的值，可以使用函数`find(k)`，该函数返回的是指向该元素的迭代器。

```c++
   	map<int, int>::iterator it_find;
    it_find = mp.find(0);
    if (it_find != mp.end()){
          it_find->second = 20;
    }else{
          printf("no!\n");
    }
```

find一定会使用迭代器，所以要学会对迭代器进行初始化

` map<int, int>::iterator it_find;`

##### 删除

从map中删除元素的函数是`erase()`，该函数有如下的三种形式：

- `m.erase(k)`
- `m.erase(p)`
- `m.erase(b, e)`

第一种方法删除的是m中键为k的元素，返回的是删除的元素的个数；第二种方法删除的是迭代器p指向的元素，返回的是void；第三种方法删除的是迭代器b和迭代器e范围内的元素，返回void。

```c++
	map<int, int> mp;
    for (int i = 0; i < 20; i++){
         mp.insert(make_pair(i, i));
    }
    mp.erase(0);
    mp.erase(mp.begin());
```

##### map常用函数

```c++
map<char,int> mymap;
mymap.insert(make_pair('a', 100));
mymap['b'] = 100;
mymap.insert(pair<char,int>('c',300));
//返回迭代器到开始(公共成员函数)
mymap.begin() 		

//返回迭代器到末尾(公共成员函数)
mymap.end()		
 
//返回反向迭代器到反向开始(公共成员函数)
mymap.rbegin()	 
    
//返回反向迭代器到反向端(公共成员函数)
mymap.rend()	
    
//将const_iterator返回到开始(公共成员函数)    
mymap.cbegin()	
 
//返回const_iterator末尾(公共成员函数)
mymap.cend()			

//返回const_reverse_iterator到反向开始(公共成员函数)    
mymap.crbegin()		

//返回const_reverse_iterator到reverse end(公共成员函数)
mymap.crend()			
 
//返回映射中是否为空    
mymap.empty()			
 
//返回映射容器可以容纳的元素的最大数量    
mymap.max_size()		

//访问元素(公共成员函数)    
mymap['a'] 		

//修改Key值的引用值    
mymap.at('a')=10;        
 
//插入元素(公共成员函数)   
mymap.insert()		
 
//删除元素(公共成员函数)    
mymap.erase()			
 
//交换内容(公共成员函数)，将两个map中的内容完全交换   
mymap.swap(mymap2)			
  
//从map容器中删除所有元素，使容器的大小为0   
mymap.clear()			    
```



#### 2.unordered_map

map内部实现的是一个黑红树，具有自动排序功能，因此map所有元素都是有序的，红黑树的每一个节点都代表着map的一个元素。因此，对于map进行的查找，删除，添加等一系列的操作都相当于是对红黑树进行的操作。map中的元素是按照二叉搜索树（又名二叉查找树、二叉排序树，特点就是左子树上所有节点的键值都小于根节点的键值，右子树所有节点的键值都大于根节点的键值）存储的，使用中序遍历可将键值按照从小到大遍历出来。

unordered_map内部实现了一个哈希表（也叫散列表，通过把关键码值映射到Hash表中一个位置来访问记录，查找的时间复杂度可达到O(1)，其在海量数据处理中有着广泛应用）。因此，其元素的排列顺序是无序的。



在有对于顺序要求的问题中，map会更高效，但是map的每个节点占用空间更大。

而对于查找问题来说，unordered_map会更加高效，因为哈希表查找速度非常快，但是哈希表建立比较消耗时间。

其操作与map相同。



#### multimap 

mutimap容器一样保存的是有序的键值对，但是与map不同的是他可以保存**重复的元素**，multimap 也同样是一颗红黑树，会将插入的键值自动排序



#### 3.vector

vector是向量类型，可以容纳许多类型的数据，因此也被称为容器，

##### vector初始化：

**方式1.**

```c
//定义具有10个整型元素的向量（尖括号为元素类型名，它可以是任何合法的数据类型），不具有初值，其值不确定
vector<int>a(10);
12
```

**方式2.**

```c
//定义具有10个整型元素的向量，且给出的每个元素初值为1
vector<int>a(10,1);
12
```

**方式3.**

```c
//用向量b给向量a赋值，a的值完全等价于b的值
vector<int>a(b);
12
```

**方式4**.

```c
//将向量b中从0-2（共三个）的元素赋值给a，a的类型为int型
vector<int>a(b.begin(),b.begin+3);
12
```

**方式5.**

```c
 //从数组中获得初值
int b[7]={1,2,3,4,5,6,7};
vector<int> a(b,b+7）;
```

##### 常用内置函数

```c
#include<vector>
vector<int> a,b;
//b为向量，将b的0-2个元素赋值给向量a
a.assign(b.begin(),b.begin()+3);

//a含有4个值为2的元素
a.assign(4,2);

//返回a的最后一个元素
a.back();

//返回a的第一个元素
a.front();

//返回a的第i元素,当且仅当a存在
a[i];

//清空a中的元素
a.clear();

//判断a是否为空，空则返回true，非空则返回false
a.empty();

//删除a向量的最后一个元素
a.pop_back();

//删除a中第一个（从第0个算起）到第二个元素，也就是说删除的元素从a.begin()+1算起（包括它）一直到a.begin()+3（不包括它）结束
a.erase(a.begin()+1,a.begin()+3);

//在a的最后一个向量后插入一个元素，其值为5
a.push_back(5);

//在a的第一个元素（从第0个算起）位置插入数值5,
a.insert(a.begin()+1,5);

//在a的第一个元素（从第0个算起）位置插入3个数，其值都为5
a.insert(a.begin()+1,3,5);

//b为数组，在a的第一个元素（从第0个元素算起）的位置插入b的第三个元素到第5个元素（不包括b+6）
a.insert(a.begin()+1,b+3,b+6);

//返回a中元素的个数
a.size();

//返回a在内存中总共可以容纳的元素个数
a.capacity();

//将a的现有元素个数调整至10个，多则删，少则补，其值随机
a.resize(10);

//将a的现有元素个数调整至10个，多则删，少则补，其值为2
a.resize(10,2);

//将a的容量扩充至100，
a.reserve(100);

//b为向量，将a中的元素和b中的元素整体交换
a.swap(b);
下标访问是之恶能访问
//b为向量，向量的比较操作还有 != >= > <= <
a==b;
```

##### 注意事项

通过下标访问时只能访问存在的元素，

##### 几个常用的算法

```c
 #include<algorithm>
 
//对a中的从a.begin()（包括它）到a.end()（不包括它）的元素进行从小到大排列
 sort(a.begin(),a.end());
 
//对a中的从a.begin()（包括它）到a.end()（不包括它）的元素倒置，但不排列，如a中元素为1,3,2,4,倒置后为4,2,3,1
 reverse(a.begin(),a.end());
 
//把a中的从a.begin()（包括它）到a.end()（不包括它）的元素复制到b中，从b.begin()+1的位置（包括它）开始复制，覆盖掉原有元素
 copy(a.begin(),a.end(),b.begin()+1);
 
//在a中的从a.begin()（包括它）到a.end()（不包括它）的元素中查找10，若存在返回其在向量中的位置
  find(a.begin(),a.end(),10);
```

#### 4.queue

只能访问queue<T>容器适配器的第一个和最后一个元素，只能在容器的末尾添加新元素，只能从头部移除元素。

queue的基本操作如下图：

![](CCFtab/queue.jpg)

queue 和 stack 有一些成员函数相似，但在一些情况下，工作方式有些不同：

```c++
queue<string>q;
//返回 queue 中第一个元素的引用。如果 queue 是常量，就返回一个常引用；如果 queue 为空，返回值是未定义的。
q.front()：

//返回 queue 中最后一个元素的引用。如果 queue 是常量，就返回一个常引用；如果 queue 为空，返回值是未定义的。
q.back()：

//在 queue 的尾部添加一个元素的副本。这是通过调用底层容器的成员函数 push_back() 来完成的。    
q.push(const T& obj)：

//以移动的方式在 queue 的尾部添加元素。这是通过调用底层容器的具有右值引用参数的成员函数 push_back() 来完成的。    
q.push(T&& obj)：

//删除 queue 中的第一个元素。
q.pop()：

//返回 queue 中元素的个数。
q.size()：

//如果 queue 中没有元素的话，返回 true。
q.empty()：

//用传给 emplace() 的参数调用 T 的构造函数，在 queue 的尾部生成对象。
emplace()：

//将当前 queue 中的元素和参数 queue 中的元素交换。它们需要包含相同类型的元素。也可以调用全局函数模板 swap() 来完成同样的操作。
swap(queue<T> &other_q)：
```

#### 5.stack

```c++
stack<int>s1;
int a=1;
//堆栈为空则返回真
s1.empty() 

//移除栈顶元素
s1.pop() 

//在栈顶增加元素a
s1.push(a) 

//返回栈中元素数目
s1.size() 

//返回栈顶元素
s1.top() 
```

#### 6.set

set容器是用来存储同一数据类型的数据类型，并能从一个数据集合取出数据，在set中每个元素的值都唯一，系统能根据元素的值自动排序，应注意set中的每个值都不能直接被改变。与map相同，set的内部同样实现的是黑红树。

##### 常用方法

```c++
//将key_value插入到set中 ，返回值是pair<set<int>::iterator,bool>，bool标志着插入是否成功，而iterator代表插入的位置，若key_value已经在set中，则iterator表示的key_value在set中的位置。
insert(key_value);

//将定位器first到second之间的元素插入到set中，返回值是void.
inset(first,second);

//返回set容器的第一个元素
begin()     　　 

//返回set容器的最后一个元素
end() 　　　　 

//删除set容器中的所有的元素
clear()   　　     

//判断set容器是否为空
empty() 　　　

//返回set容器可能包含的元素最大个数
max_size() 　 

//返回当前set容器中的元素个数
size() 　　　　 

//返回的值和end()相同
rbegin　　　　 

//返回的值和rbegin()相同
rend()　　　　 

//返回一对定位器，分别表示第一个大于或等于给定关键值的元素和 第一个大于给定关键值的元素，这个返回值是一个pair类型    
equal_range() 

//删除定位器iterator指向的值
erase(iterator)  

//删除定位器first和second之间的值
erase(first,second)

//删除键值key_value的值
erase(key_value)

//返回给定值值得定位器，如果没找到则返回end()。
find() 

//返回第一个大于等于key_value的定位器
lower_bound(key_value)

//返回最后一个大于key_value的定位器
upper_bound(key_value)
```

### 常用函数原型以及解释

以下函数以及头文件均包含在bits/stdc++.h中，无需特殊声明。

#### `#include<ctype.h>`

这几个都是返回非0表示正确，返回0表示错误

1.`isalnum(int c);`

​	用来判断一个字符是否是字母或者十进制数字。其中c表示要检测的字符，（被转化为int类型，可以直接输入char）也可以是EOF 



2.`islower(int c);`

​	用来检测一个字符是不是小写字母，其中c表示要检测的字符，（被转化为int类型，可以直接输入char）也可以是EOF 



3.`isalpha(int c);`

​	用来检测一个字符是不是字母，其中c表示要检测的字符，（被转化为int类型，可以直接输入char）也可以是EOF 



4.`isdigit(int c);`

​	用来检测一个字符是不是数字，其中c表示要检测的字符，（被转化为int类型，可以直接输入char）也可以是EOF 



5.`isblank(int c);`

​	用来检测一个字符是不是空白符，其中c表示要检测的字符，这个函数仅检测`空格‘’`,`水平制表符‘/t’`，如果想检测更多的话需要用函数`isspace()`;



6.`iscntrl(int c);`

​	iscntrl() 函数用来检测一个字符是否是控制字符（Control Character）。控制字符的范围是`0x00 (NUL) ~ 0x1f (US)`，再加上一个`0x7f (DEL)`字符。



7.`isgraph(int c); `

​	isgraph() 函数用来检测一个字符是否是图形字符。



8.`ispunct(int c);`

​	ispunct() 函数用来检测一个字符是否是标点符号。



9.`isupper ( int c );`

​	isupper() 函数用来检测一个字符是否是大写字母。



10.`isxdigit ( int c );`

​	isxdigit() 用来检测一个字符是否是十六进制数字。



11.`tolower ( int c );`

​	tolower() 函数用来将大写字母转换为小写字母。



12.toupper ( int c );

​	toupper() 函数用来将小写字母转换为大写字母。



#### `#include<algorithm>`

1.`reverse(it,it2)`

​	**将数组指针在[it,it2)之间的元素或容器的迭代器在[it,it2)范围内的元素进行反转。string也可以。**

```c++
int a[10]= {10, 11, 12, 13, 14, 15}; 
reverse(a, a+4);//a[0]~a[3]反转

string str = "abcdefghi";
reverse(str.begin()+2, str.begin()+6);//对a[2]~a[5]逆转*左闭右开* 
```

2.`sort(a,a+k,cmp)`

​	**sort(首元素地址(必填), 尾元素地址的下一个地址(必填), 比较函数(非必填));不填写时默认为递增序列。**

```c++
int a[6] = {9, 4, 2, 5, 6, -1};
//将a[0]~a[3]从小到大排序
sort(a, a+4);

//cmp函数用法
bool cmp(int a, int b)
{
	return a > b;  //可以理解为当a>b时把a放在b前面 
}
```

#### `#include<string>`

1.`void *memchr(const void *str, int c, size_t n)`

在参数str指向的字符串的前n个字节中搜索第一次出现字符c（无符号字符）的位置

```c
#include <stdio.h>
#include <string.h>
 
int main ()
{
   const char str[] = "http://www.runoob.com";
   const char ch = '.';
   char *ret;
 
   ret = (char*)memchr(str, ch, strlen(str));
 
   printf("|%c| 之后的字符串是 - |%s|\n", ch, ret);
 
   return(0);
}
```

2.`int memcmp(const void*str1,const void*str2,size_t n);`

将str1和str2的前n个字节进行比较

字符串相等返回0，字符串前大于后返回1否则返回0



3.`void *memcpy(void *dest, const void *src, size_t n)`
从 src 复制 n 个字符到 dest。



4.`void *memmove(void *dest, const void *src, size_t n)`
另一个用于从 src 复制 n 个字符到 dest 的函数。



5.`void *memset(void *str, int c, size_t n)`
复制字符 c（一个无符号字符）到参数 str 所指向的字符串的前 n 个字符。



6.`char *strcat(char *dest, const char *src)`
把 src 所指向的字符串追加到 dest 所指向的字符串的结尾。

该函数返回一个指向最终的目标字符串 dest 的指针。



7.`char *strncat(char *dest, const char *src, size_t n)`
把 src 所指向的字符串追加到 dest 所指向的字符串的结尾，直到 n 字符长度为止。



8.`char *strchr(const char *str, int c)`
在参数 str 所指向的字符串中搜索第一次出现字符 c（一个无符号字符）的位置。



9.`int strcmp(const char *str1, const char *str2)`
把 str1 所指向的字符串和 str2 所指向的字符串进行比较。

该函数返回值如下：

- 如果返回值小于 0，则表示 str1 小于 str2。
- 如果返回值大于 0，则表示 str1 大于 str2。
- 如果返回值等于 0，则表示 str1 等于 str2。



10.`int strncmp(const char *str1, const char *str2, size_t n)`
把 str1 和 str2 进行比较，最多比较前 n 个字节。



11.`int strcoll(const char *str1, const char *str2)`
把 str1 和 str2 进行比较，结果取决于 LC_COLLATE 的位置设置。



12.`char *strcpy(char *dest, const char *src)`

把 src 所指向的字符串复制到 dest。



13.`char *strncpy(char *dest, const char *src, size_t n)`
把 src 所指向的字符串复制到 dest，最多复制 n 个字符。



14.`size_t strcspn(const char *str1, const char *str2)`
检索字符串 str1 开头连续有几个字符都不含字符串 str2 中的字符。

该函数返回 str1 开头连续都不含字符串 str2 中字符的字符数。

以匹配到第一个字符为准。



15.`char *strerror(int errnum)`
从内部数组中搜索错误号 errnum，并返回一个指向错误消息字符串的指针。strerror生成的错误字符串取决于开发平台和编译器。



16.`size_t strlen(const char *str)`
计算字符串 str 的长度，直到空结束字符，但不包括空结束字符。



17.`char *strpbrk(const char *str1, const char *str2)`
检索字符串 str1 中第一个匹配字符串 str2 中字符的字符，不包含空结束字符。

该函数返回 str1 中第一个匹配字符串 str2 中字符的字符数，如果未找到字符则返回 NULL。

```c
const char str1[] = "abcde2fghi3jk4l";
   const char str2[] = "34";
   char *ret;
 
   ret = strpbrk(str1, str2);
   if(ret) 
   {
      printf("第一个匹配的字符是： %c\n", *ret);
   }
   else 
   {
      printf("未找到字符");
   }
   
//返回3
```



18.`char *strrchr(const char *str, int c)`
在参数 str 所指向的字符串中搜索最后一次出现字符 c（一个无符号字符）的位置。



19.`size_t strspn(const char *str1, const char *str2)`
检索字符串 str1 中第一个不在字符串 str2 中出现的字符下标。



20.`char *strstr(const char *haystack, const char *needle)`
在字符串 haystack 中查找第一次出现字符串 needle（不包含空结束字符）的位置。



21.`char *strtok(char *str, const char *delim)`
分解字符串 str 为一组字符串，delim 为分隔符。



22.`size_t strxfrm(char *dest, const char *src, size_t n)`
根据程序当前的区域选项中的 LC_COLLATE 来转换字符串 src 的前 n 个字符，并把它们放置在字符串 dest 中。

### 运用文件读写加快测试速度的代码

```c++
#include<bits/stdc++.h>
using namespace std;
int main(){
    int a,b;
    freopen("in.txt","r",stdin);
    /*
    if(freopen("in.txt","r",stdin)==NULL)
    	fprintf(stderr,"errorredirecting stdout\n")
    */
    freopen("out.txt","w",stdout);
    cin>>a;
    cin>>b;
    cout<<a+b<<endl;
    fclose(stdin);
    fclose(stdout);
    return 0;
}
```

在创建.cpp文件的地方创建in.txt文件和out.txt文件

