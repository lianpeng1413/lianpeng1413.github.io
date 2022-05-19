## STL Overview

# STL Overview

STL（Standard Template Library）是 C++ 泛型编程（Generic Programming）的体现，将算法从数据结构中抽象出来，以相同或相近的方式处理各种不同情形。


![stl-overview.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652855647382/LberqF98N.png align="left")

其组件分为六大类：

* Allocator（分配器），负责空间的配置与管理，是一个实现了动态空间配置、空间管理、空间释放的 class template；
* Container（容器），各种数据结构，如 `vector`, `list`, `deque`, `set`, `map` 用来存放数据；
* Adapter（适配器），可改变 containers、Iterators 或 Functors 接口的东西，例如 `queue` 和 `stack` ；
* Algorithm（算法），各种基本算法如 sort、search 等
* Iterator（迭代器），连接 containers 和 algorithms
* Functors（仿函数），是一种重载了 operator() 的 class 或 class template。

## Allocator

以下介绍 `std::alloc`。

### 空间的配置与释放

对象构造前的空间配置和对象析构后的空间释放，由 <stl_alloc.h> 负责：

* 向 system heap 要求空间；
* 考虑多线程（multi-threads）状态；
* 考虑内存不足时的应变措施；
* 考虑过多内存碎片问题。

当前先不考虑多线程状态的处理。

SGI 设计了双层级 allocator，第一级 allocator 直接使用 malloc() 和 free() ，第二级 allocator 维护需要申请小于 128 B 的内存块。

![stl-allocator.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652850686341/s0TdA-WwN.png align="left")


#### 第一级 allocator

* 其中 allocate 和 deallocate 分别直接使用 malloc 和 free；
* 模拟 C++ 的 set_new_handler() 以处理内存不足的情况。

需要定义 oom 相关函数指针，以处理内存不足，并允许客户端能够设置自己的 handler。

内存不足处理程序是客户端的责任，不应该自己随便处理。

#### 第二级 allocator

主要函数如下：

```c++
static void *refill(size_t n); // 用于配置一大块空间，可容纳 nobjs（默认 <=20）个大小为 n 的内存块
static char *chunk_alloc(size_t size, int &nobjs);

// Chunk allocation state
static char *start_free; // 内存池起始位置，只在 chunk_alloc() 中变化（内存池工作）
static char *end_free; // 内存池结束位置，只在 chunk_alloc() 中变化（内存池工作）
static size_t heap_size;

public:
  static void * allocate(size_t n);
  static void deallocate(void *p, size_t n);
  static void * reallocate(void *p, size_t old_sz, size_t new_sz);
};
```



* 如果所需的内存块够大，超过 128 B，交给上一级 allocator 处理；

* 如果内存块小于 128 B，应由内存池管理，即次层 allocate；

  * （ROUND_UP）主动将用户所需要的内存大小上调至 8 的倍数，并维护 16 个 free-lists，管理 8, 16, 24 ~ 128 B 的内存块，free-lists 的节点结构为：

    ```c++
    union obj {
      union obj * free_list_link; // 下一个块的内存地址
      char client_data[8]; // 用户所需要的内存空间
    }
    ```

    实现了[一物两用](https://blog.csdn.net/u014587123/article/details/81628151)的目的。

  * 空间配置函数 allocate()

    ```c++ 
    static void * allocate(size_t n) {
      obj * volatile * my_free_list;
      obj * result;
      
      // 大于 128 B 调用第一级 allocator
      if (n > (size_t) __MAX_BYTES) {
        return (malloc_alloc::allocate(n));
      }
      // 寻找 16 个 free_list 中适当的一个
      my_free_list = free_list + FREELIST_INDEX(n);
      result = my_free_list;
      if (result == 0) {
        // 没找到可用的 free list，重新填充 free list
        void *r = refill(ROUND_UP(n));
        return r;
      }
      *my_free_list = result->free_list_link;
      return (result);
    }
    ```

    其中 refill 用来重新填充 free list：

    ```c++
    template <bool threads, int inst>
    void* __default_alloc_template<threads, inst>::refill(size_t n) {
      int nobjs = 20; // 默认取得 20 个内存块
      // 调用 chunk_alloc()，取得 nobjs 个内存块作为 free list 新节点
      // nobjs 是 pass by reference
      char * chunk = chunk_alloc(n, nobjs);
      obj * volatile * my_free_list;
      obj * result;
      obj * current_obj, * next_obj;
      int i;
      
      // 如果只获取一个 chunk，直接分配给调用者
      if (1 == nobjs) return (chunk);
      // 否则将空闲 chunk 加入到 free list 里
      my_free_list = free_list + FREELIST_INDEX(n);
      
      // 在 chunk 空间内建立 free list
      result = (obj *) chunk; //返回给客户端一个
      *my_free_list = next_obj = (obj *)(chunk + n);
      for (i = 1; ; i++) { // 从 1 开始
        current_obj = next_obj;
        next_obj = (obj *)((char *)next_obj + n);
        if (nobjs - 1 == i) {
          current_obj->free_list_link = 0;
          break;
        } else {
          current_obj->free_list_link = next_obj;
        }
      }
      return (result);
    }
    ```

  * 空间释放函数 deallocate()，相当于把回收的空间放到链表的第一项。

    ```c++
    static void deallocate(void *p, size_t n) {
      obj * q = (obj *)p;
      obj * volatile * my_free_list;
      
      // 大于 128 B 调用第一级配置器
      if (n > (size_t) __MAX_BYTES) {
        malloc_alloc::deallocate(p, n);
        return;
      }
      // 寻找对应的 free list
      my_free_list = free_list + FREELIST_INDEX(n);
      // 调整 free list，回收区块
      q->free_list_link = *my_free_list;
      *my_free_list = q;
    }
    ```

  * 内存池 memory pool 的 chunk_alloc()

    ```c++
    template <bool threads, int inst>
    char * __default_alloc_template<threads, inst>::chunk_alloc(size_t size, int& nobjs) {
      char * result;
      size_t total_bytes = size * nobjs;
      size_t bytes_left = end_free - start_free; //内存池剩余空间
      
      if (bytes_left >= total_bytes) {
        // 内存池剩余空间足够
        result = start_free;
        start_free += total_bytes;
        return (result);
      } else if (bytes_left >= size) {
        // 内存池剩余空间不够 20 个，但足够一个以上的内存块
        nobjs = bytes_left / size;
        total_bytes = size * nobjs;
        result = start_free;
        start_free += total_bytes;
        return (result);
      } else {
        // 内存池剩余空间不够一个区块的大小
        size_t bytes_to_get = 2 * total_bytes + ROUND_UP(heap_size >> 4); // ?
        if (bytes_left > 0) {
          // 内存池剩余部分需要放到 free list 里
          obj * volatile * my_free_list = free_list + FREELIST_INDEX(bytes_left);
          ((obj *)start_free)->free_list_link = *my_free_list;
          *my_free_list = (obj *)start_free;
        }
        
        // 配置 heap 空间，补充内存池
        start_free = (char *)malloc(bytes_to_get);
        if (0 == start_free) {
          // heap 空间不足，malloc 失败
          int i;
          obj * volatile * my_free_list, *p;
          //搜索 free list 里，比当前内存块更大的块链表，
          for (i = size; i <= __MAX_BYTES; i+= __ALIGN) {
            my_free_list = free_list + FREELIST_INDEX(i);
            p = *my_free_list;
            if (0 != p) {
              *my_free_list = p->free_list_link;
              start_free = (char *)p;
              end_free = start_free + i;
              // 递归调用自己修正 nobjs
              return (chunk_alloc(size, nobjs));
              // 残余的零头将被编入适当的 free list
            }
          }
          // 如果到处都没内存可用了
          end_free = 0; 
          // 调用第一级 allocator
          start_free = (char *)malloc_alloc::allocate(bytes_to_get);
        }
        // 如果 malloc 成功
        heap_size += bytes_to_get;
        end_free = start_free + bytes_to_get;
        // 递归调用自己，修正 nobjs
        return (chunk_alloc(size, nobjs));
      }
    }
    ```

这里的流程可以用这张图片来描述（[来源](https://www.cnblogs.com/LUO77/p/5824625.html)于此）：

![stl-allocator-flow.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1652851235328/b20TD3PjP.jpeg align="left")

### 内存基本处理工具

STL 定义有五个全局函数。

#### construct()

```c++
#include <new.h>

// construct 函数
template <class T1, class T2>
inline void construct(T1* p, const T2& value) {
  new (p) T1(value); // placement new; 调用 T1::T1(value);
}
```

#### destroy()

destroy 函数需要区分所要 destroy 的 value_type 是否是 non-trivial destructor（不是微不足道的析构函数）。这样做的目的是，在大范围析构时，可以只一次次调用那些 non-trivial destructor，提高效率。其中 value_type() 和 __type_traits<> 将在 3.7 节介绍。

```c++
// destroy 函数，第一个版本，只接受一个指针
template <class T>
inline void destroy(T* pointer) {
  pointer->~T();
}

// destroy 函数，第二个版本，接受两个迭代器，设法找出元素的 value type，进而利用 __type_traits<> 找最合适的方法
template <class ForwardIterator>
inline void destroy(ForwardIterator first, ForwardIterator last) {
  __destroy(first, last, value_type(first));
}

// 判断元素的 value type 是否有 trivial destructor
// trivial destructor: value type 的析构函数没有被显式定义，说明“微不足道”
template <class ForwardIterator, class T>
inline void __destroy(ForwardIterator first, ForwardIterator last, T*) {
  typedef typename __type_traits<T>::has_trivial_destructor trivial_destructor;
  __destroy_aux(first, last, trivial_destructor);
}

// 如果有 non-trivial destructor
template <class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last, __false_type) {
  for (; first < last; ++first) {
    destroy(&*first);
  }
}

// 如果没有 non-trivial destructor
template <class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last, __true_type) {}
```

#### uninitialized_copy

```c++
template <class InputIterator,class ForwardIterator>
ForwardIterator uninitialized_copy(InputIterator first, InputIterator last, ForwardIterator result);
```

将内存配置与对象构造行为分离开，如果范围在 [result, result + (last - first)) 的每个迭代器所指向的区域未被初始化，uninitialized_copy 会使用 copy constructor，给 [first, last) 的每个对象产生一份复制品，放进输出范围中。即：

```c++
for (auto i = first; i < last; i++) {
  construct(&*(result + (i - first)), *i);
}
```

如果需要实现一个容器，其全区间构造函数可以分为以下两个步骤完成：

* 配置内存块，包含范围内的所有元素；
* 使用 `uninitialized_copy()`，在该内存区块上构造元素。

标准要求 uninitialized_copy() 具有 commit or rollback 语义，要么构造所有必要元素，要么什么也不构造。

#### uninitialized_fill

```c++
template <class ForwardIterator, class T>
void uninitialized_fill(ForwardIterator first, ForwardIterator last, const T& x);
```

如果 [first, last) 范围内的每个迭代器都指向未初始化的内存：

```c++
for (auto i = first; i < last; i++) {
  construct(&*i, x);
}
```

标准要求 uninitialized_fill 具有 commit or rollback 语义，要么全部构造成功， 要么全部没有构造。

#### uninitialized_fill_n

```c++
template <class ForwardIterator, class Size, class T>
void uninitialized_fill(ForwardIterator first, Size n, const T& x);
```

为指定范围 [first, first + n) 内的所有元素设定相同的初值。

类似于 destroy 需要讨论 value type 是否具备 non-trivial destructor，unitialized* 函数也需要判断 value type 是否是 POD 类型。

> POD 即 Plain Old Data，是 scalar types 或 C struct 类型。
>
> POD 类型一定有 trivial constructor / destructor / copy / assignment 函数。
>
> 对于 POD 类型，直接交由高阶函数进行（详见 6.4.2 《STL 源码剖析》），调用 STL 的 `copy` / `fill` 函数
>
> 对于非 POD 类型，逐个调用 `construct` 函数构造。

##### 如何判断 POD 类型

以 copy 为例。

```c++
template <class InputIterator, class ForwardIterator, class T>
inline ForwardIterator __uninitialized_copy(InputIterator first, InputIterator last, ForwardIterator result, T*) {
  typedef typename __type_traits<T>::is_POD_type is_POD;
  return __uninitialized_copy_aux(first, last, result, is_POD());
  // 利用 is_POD() 获得的结果，由编译器进行参数推导。
}

```

##### char \*/ wchar_t \* 特化

针对 char \* 和 wchar_t \* 可以采用 `memmove` 直接移动内存内容来执行复制行为。

## Iterator 迭代器以及 traits

是 container 和 algorithm 的粘合剂。

迭代器最重要的工作是重载 `operator *` 和 `operator ->`。可以参考 `auto_ptr` ，一个包装原生指针的对象，能够解决内存泄漏问题。

在这一部分，核心思想是通过编译来让编译器自行确定类别和相关函数。

### 迭代器类型

* Input iterator：不允许外界改变，只读的迭代器；
* Output iterator：只写的迭代器；
* Forward iterator：允许“写入型”算法，如 `replace()`、`sort()`，在此种迭代器所形成的区间上进行读写操作；
* Bidirectional iterator：可双向移动；
* Random Access iterator：支持随机访问。

设计算法时，尽量使用最贴切的迭代器以提升效率。


![stl-iterator-types.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652862982266/TeSbHbBj1.png align="left")

### traits 萃取 associate types

![stl-iterator-traits.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652861315472/7jDf4OwBY.png align="left")

```c++
template <class I>
struct iterator_traits { // traits 特性
  typedef typename I::value_type value_type; // 例如 list<int>::iterator
  typedef typename I::iterator_category iterator_category;
  typedef typename I::difference_type difference_type;
  typedef typename I::pointer pointer;
  typedef typename I::reference reference;
}

template <class T>
struct iterator_traits<T*> { 
  typedef T value_type; // 例如 int*
  typedef ptrdiff_t difference_type; // 作为原生指针的 difference type
  typedef T* pointer;
  typedef T& reference;
  typedef random_access_iterator_tag iterator_category; // 原生指针是一种 random access iterator
};

template <class T>
struct iterator_traits<const T*> {
  typedef T value_type;
  typedef ptrdiff_t difference_type; // 作为原生 const 指针的 difference type
  typedef const T* pointer;
  typedef const T& reference;
  typedef random_access_iterator_tag iterator_category;
}
```

#### value type

指的是迭代器所指对象的类型。

#### difference type

表示两个迭代器之间的距离，也可以用来表示一个连续空间容器的最大容量，例如 STL 的 `count()`:

```c++
template <class I, class T>
typename iterator_traits<I>::difference_type count(I first, I last, T& value) {
  typename iterator_traits<I>::difference_type n = 0;
  for (; first != last; first++) {
    if (*first == value) {
      ++n;
    }
  }
  return n;
}
```

#### reference type

如果 iterator 是 p，那 reference type 指的是 `*p`。

| 迭代器             | reference type |
| ------------------ | -------------- |
| constant iterators | const T&       |
| mutable iterators  | T&             |

#### pointer type

T* 是 pointer type，指向所指之物的地址。

#### iterator_category

迭代器类型，有五种。注意，原生指针是一种 random access iterator。



## Sequence Containers

![stl-container.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652876891214/TJZ_SAywx.png align="left")

### vector

向量是一个项的有限序列，满足：

1. 给出序列中的任何下标，能够在常数时间内访问或修改下标对应的项；
2. 在序列尾部进行的插入，平均时间复杂度为常数，但是最坏时间复杂度为\\(O(n) \\) ，其中 n 为项的数量；
3. 对序列尾部进行删除，最坏时间复杂度为常数；
4. 对任意的插入删除，平均/最坏时间复杂度均为 \\(O(n) \\)

vector 类的两个模板参数有：

```c++
template<class T, class Allocator=allocator>
// T 代表 entry 的类型
// allocator 是分配器，默认使用 <defalloc> 中的缺省分配模型
```

#### 常用方法接口


| 方法                       | 功能                                                         |
| -------------------------- | ------------------------------------------------------------ |
| vector\<double\> weights   | 初始化一个空向量                                             |
| weights.push_back(107.2)   | 尾部插入，通过拷贝构造/移动构造函数                          |
| weights.insert(itr, 125.0) | 在 itr 所在位置插入一个 125.0；<br />weights 中插入点之后的项依次向后移动一个单位，返回位于新插入项的迭代器 |
| weights.pop_back()         | 删除尾部项                                                   |
| weights.erase(itr)         | 删除 itr 所在位置的项；<br />高位项依次向前移动；<br />令所有指向擦除位置的后面位置迭代器失效 |
| weights.size()             | 返回项数量                                                   |
| weights.empty()            | 如果为空返回真，否则返回 false                               |
| weights.front() = 105.0    | 将 weights 下标 0 处的项替换成 105.0                         |
| weights[3] = 110.5         | 将 weights 中下标 3 处的项替换为 110.5                       |

#### vector 定义摘要

```c++
template <class T, class Alloc = alloc>
class vector {
  public:
  	// vector 的类型定义
    typedef T value_type;
    typedef value_type* pointer;
    typedef value_type* iterator; // 可以使用普通指针作为迭代器
    typedef value_type& reference;
    typedef size_t size_type;
    typedef ptrdiff_t difference_type;
  protected:
  	typedef simple_alloc<value_type, Alloc> data_allocator;
  	iterator start; // 表示当前使用空间的头
  	iterator finish; // 表示当前使用空间的尾
  	iterator end_of_storage; // 表示当前可用空间的尾
  	void insert)aux(iterator position, const T& x);
  	void deallocate() {
      if (start) {
        data_allocator::deallocate(start, end_of_storage - start);
      }
    }
 
  	void fill_initialize(size_type n, const T& value) {
      start = allocate_and_fill(n, value);
      finish = start + n;
      end_of_storage = finish;
    }
  public:
  	iterator begin() { return start; }
  	iterator end() { return finish; }
    size_type size() const { return size_type(end() - begin(); )}
  	size_type capacity() const {
      return size_type(end_of_storage - begin());
    }
  	bool empty() const { return begin() == end(); }
  	reference operator[](size_type n) { return *(begin + n); }
  
  	vector(): start(0), finish(0), end_of_storage(0) {}
  	vector(size_type n, const T& value) { fill_initialize(n, value); }
  	vector(int n, const T& value) { fill_initialize(n, value); }
  	vector(long n, const T& value) { fill_initialize(n, value); }
  	explicit vector(size_type n) { fill_initialize(n, T()); }
  
  	~vector() {
      destroy(start, finish); // 调用析构函数
      deallocate(); // 释放内存空间
    }
  
  	reference front() { return *begin(); }
  	reference back() { return *(end() - 1); }
  	void push_back(const T& x) {
      if (finish != end_of_storage) {
        construct(finish, x);
        ++finish;
      } else {
        insert_aux(end(), x);
      }
    }
  
  	void pop_back() {
      --finish;
      destroy(finish);
    }
  
  	iterator erase(iterator position) {
      if (position + 1 != end()) {
        copy(position + 1, finish, position);
      }
      --finish;
      destroy(finish);
      return position;
    }
  
  	void resize(size_type new_size, const T& x) {
      if (new_size < size()) {
        erase(begin + new_size(), end());
      } else {
        insert(end(), new_size - size(), x);
      }
    }
  
  	void resize(size_type new_size) { resize(new_size, T()); }
  	void clear() { erase(begin(), end()); }
  
  protected:
  	iterator allocate_and_fill(size_type n, const T& x) {
      iterator result = data_allocator::allocate(n);
      uninitialized_fill_n(result, n, x);
      return result;
    }
}
```



#### 一些实现细节

1. vector 是如何存储的？

   当 vector\<double\> weights 为空时，调用第一个 push_back，此时将分配堆中的一个存储块（随编译器不同而异），如果这一块是 1024B，将分配一个 128 个项的 double 型数组。此时 weights.end() 指向 weights[1]，`weights.end_of_storage` 指向数组之后的第一个单元。

   如果一个 vector 对象对应的数组已满，而且又尝试新的插入，将重新申请一个新的堆存储块（原来项的数量 * 2）。旧的项将被拷贝到新的数组，旧数组的存储空间被回收，然后将新的项插入新数组。

   通过扩容 vector 数组，来减少 push_back 时导致的频繁内存移动。

2. insert 的实现细节，注意如何进行内存申请，以及插入过程。时间复杂度为 \\(O(n) \\)

   ```c++
   iterator insert(iterator position, const T& x) {
     size_type n = position - begin();
     if (finish != end_of_storage) { // 当前数组没有被填充满
       if (position == end()) {
         construct(finish, x); // 如果当前位置正好是尾部则直接构造插入
         finish++;
       } else {
         construct(finish, *(finish - 1));
         copy_backward(position, finish - 1, finish);
         *position = x;
         ++finish;
       }
     } else {
       // 当前数组已经被填充满，或者该数组还没有被分配空间
       // 如果已经被填充满，需要扩容至原来的 2 倍
       // 如果没有被分配空间，需要先分配一个 page_size 作为当前的内存的 len
       size_type len = size() ? 2 * size() : static_allocator.init_page_size();
       iterator tmp = static_allocator.allocate(len);
       uninitialized_copy(begin(), position, tmp);
       construct(tmp + (position - begin()), x);
       uninitialized_copy(position, end(), tmp + (position - begin()) + 1);
       destroy(begin(), end()); // 为向量的每一项调用析构器
       static_allocator.deallocate(begin());
       end_of_storage = tmp + len;
       finish = tmp + size() + 1;
       start = tmp;
     }
     return begin() + n;
   }
   ```

### deque

顺序容器，双端队列是有如下特征的项的有限序列：

* 给定序列中任意项的下标，就可以花费常数时间访问或修改这个下标上的项；
* 平均情况下，在序列头或尾的插入只耗费常数时间，但是最坏时间复杂度为 \\(O(n) \\)，这里 n 代表序列中项的数量；
* 在序列尾进行的插入和删除，最坏时间复杂度为常数；
* 对于任意的插入和删除，最坏时间复杂度和平均时间复杂度为 \\(O(n) \\)。

![stl-deque.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652776529768/5KCk1P-3j.png align="left")

#### duque 迭代器

```c++
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator { // 未继承 std::iterator
  typedef __deque_iterator<T, T&, T*, BufSiz> iterator;
  typedef __deque_iterator<T, const T&, const T*, BufSiz> const_iterator;
  static size_t buffer_size() { return __deque_buf_size(BufSiz, sizeof(T)); }
  
  // 由于没有继承 std::iterator
  typedef random_access_iterator_tag iterator_category;
  typedef T value_type;
  typedef Ptr pointer;
  typedef Ref reference;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
  
  typedef T** map_pointer;
  typedef __deque_iterator self;
  
  // 保持与容器的联结
 	T* cur; // 所指缓冲区 buffer 的当前元素
  T* first; // 所指缓冲区 buffer 的头元素
  T* last; //所指缓冲区 buffer 的尾元素
  map_pointer node; // 指向 map 对应元素位置
}
```

其中 __deque_buf_size 是全局函数，定义如下：

```c++
// 如果 n 不为 0，传回 n，表示 buffer size 由用户自定义
// 如果 n 为 0，表示 buffer size 使用默认值
//		如果元素大小小于 512，返回 512 / 元素大小
//    否则，传回 1
inline size_t __deque_buf_size(size_t n, size_t sz) {
  return n != 0 ? n : (sz < 512 ? size_t(512 / sz) : size_t(1));
}
```

deque 的关键运算符重载，可能需要调用 `set_node()` 跳过一个缓冲区：

```c++
void set_node(map_pointer new_node) {
  node = new_node;
  first = *new_node;
  last = first + different_type(buffer_size());
}
```

重载的运算符，如 `+=` 、`-=`、`--`、`++`、指针、引用等：

```c++
reference operator*() const { return *cur; }
pointer operator->() const { return &(operator*()); }

// 获取 deque 两元素的距离
difference_type operator-(const self& x) const {
  return difference_type(buffer_size()) * (node - x.node - 1) + (cur - first) + (x.last - x.cur);
}

self& operator++() { //++i;
  ++cur;
  if (cur == last) {
    setnode(node + 1);
    cur = first;
  }
  return *this;
}

self operator++(int) { // i++;
  self tmp = *this;
  ++*this; // 调用上一个重载的 self& operator++()
  return tmp;
}

self& operator--() { // --i;
  if (cur == first) {
    set_node(node - 1);
    cur = last;
  }
  --cur;
  return *this;
}

self operator--(int) { // i--;
  self tmp = *this;
  --*this;
  return tmp;
}

// 实现随机存取
self& operator+=(difference_type n) {
  difference_type offset = n + (cur - first);
  if (offset >= 0 && offset < difference_type(buffer_size())) {
    // 在同一个缓冲区内
    cur += n;
	} else {
    // 不在同一个缓冲区内
    difference_type node_offset = 
      offset > 0 ? offset / difference_type(buffer_size()) : 
    					-difference_type((- offset - 1) / buffer_size()) - 1;
    // 切换至正确的节点
    set_node(node + node_offset);
    // 切换至正确的元素
    cur = first + (offset - node_offset * difference_type(buffer_size()));
  }
  return *this;
}

// More Effective C++ item22: Consider using op= instead of stand-alone op
self operator+(difference_type n) const {
  self tmp = *this;
  return tmp += n;
}

self operator-=(difference_type n) { return *this += (- n); }

self operator-(difference_type n) const {
  self tmp = *this;
  return tmp -= n;
}

reference operator[](difference_type n) const { return *(*this + n); }
bool operator==(const self& x) const { return cur == x.cur; }
bool operator!=(const self& x) const { return !(*this == x); }
bool operator<(const self& x) const {
  return (node == x.node) ? (cur < x.cur) : (node < x.node);
}
```

#### deque 实现

deque 由一段一段的定量连续空间组成，如图所示，其中 map 被称为映射数组，每一个元素指向保存项的连续存储块，**所有这些块是相同大小的**，start 和 finish 分别指向队列的第一项和最后一项之后的位置。

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0>
class deque {
  public:
  	typedef T value_type;
  	typedef value_type* pointer;
  	typedef size_t size_type;
  
  public:
  	typdef __deque_iterator<T, T&, T*, BufSiz> iterator;
  
  protected:
  	typedef pointer* map_pointer;
  
  protected:
  	iterator start;
  	iterator finish;
  
  	map_pointer map; // 指向 map，map 是一块连续空间，每个元素都是一个指针，指向一个 buffer
  	size_type map_size; // map 内的指针数量
  
  public:
  	iterator begin() { return start; }
  	iterator end() { return finish; }
  
  	reference operator[](size_type n) {
      return start[difference_type(n)]; // 调用 __deque_iterator<>::operator[]
    }
  
  	reference front() { return *start; }
  	reference back() {
      iterator tmp = finish;
      --tmp;
      return *tmp;
    }
  	size_type size() const { return finish - start; }
  	size_type max_size() const { return size_type(-1); } // ?
  	bool empty() const { return finish == start; }
}
```

#### deque 内存管理

deque 自行定义了两个专属的空间配置器：

```c++
protected:
	// 每次配置一个元素大小
	typedef simple_alloc<value_type, Alloc> data_allocator;
	// 每次配置一个指针大小
	typedef simple_alloc<pointer, Alloc> map_allocator;

// 并提供一个 constructor
 	deque(int n, const value_type& value) : start(), finish(), map(0), map_size(0) {
    fill_initialize(n, value); // 负责产生并安排好 deque 的结构，并将元素初值设置妥当
  }

	template <class T, class Alloc, size_t BufSize>
  void deque<T, Alloc, BufSize>::fill_initialize(size_type n, const value_type& value) {
    create_map_and_nodes(n); // 把 deque 的结构都产生并安排好
    map_pointer cur;
    __STL_TRY {
      // 为每一个节点的缓冲区都设置初值
      for (cur = start.node; cur < finish.node; ++cur) {
        uninitialized_fill(*cur, *cur + buffer_size(), value);
			}
      uninitialized_fill(finish.first, finish.cur, value); // 对于最后一个缓冲区，尾部可能有备用空间，不用设置初值
    } catch(...) {
      ...
    }
  }

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::create_map_and_nodes(size_type num_elements) {
  // 需要的缓冲区数 = (元素个数 / 每个缓冲区可容纳个数) + 1
  // 如果刚好整除，需要多配一个节点
  size_type num_nodes = num_elements / buffer_size() + 1;
  
  // 一个 map 要管理几个节点，最少 8 个，最多是 所需节点数 + 2
  // 前后各预留一个，扩充时可用
  map_size = max(initial_map_size(), num_nodes + 2);
  map = map_allocator::allocate(map_size);
  
  map_pointer nstart = map + (map_size - num_nodes) / 2;
  map_pointer nfinish = nstart + num_nodes - 1;
  
  map_pointer cur;
  __STL_TRY {
    // 为 map 内每个现用节点配置缓冲区，所有缓冲区加起来就是 deque 的可用空间
    for (cur = nstart; cur <= nfinish; ++cur) {
      *cur = allocate_node();
    }
  } catch(...) {
    ...
  }
  
  // 为 deque 内的两个迭代器 start 和 end 设置正确内容
  start.set_node(nstart);
  finish.set_node(nfinish);
  start.cur = start.first;
  finish.cur = finish.first + num_elements % buffer_size();
}
```



deque 提供了四种方法，可以直接对 back 和 front 进行操作：`pop_front()`、`pop_back()`、`push_front()`、`push_back()`。

以 push_back() 为例，介绍相关 map 的内存管理。

```c++
void push_back(const value_type& t) {
  if (finish.cur != finish.last - 1) {
    // 最后缓冲区尚有两个及以上的元素备用空间
    construct(finish.cur, t); // 直接在备用空间上构造元素
    ++finish.cur;							// 调整最后缓冲区的使用状态
	} else {
    // 最后缓冲区只剩下一个元素备用空间
    push_back_aux(t); // 先配置一整块新的缓冲区，再设置新元素内容，更改迭代器 finish 状态。
  }
}

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::push_back_aux(const value_type& t) {
  value_type t_copy = t;
  reserve_map_at_back(); // 如果 map 尾端节点备用空间不足，需要重换一个 map
												 // 这里将调用 reallocate_map(nodes_to_add, false);
  *(finish_node + 1) = allocate_node(); // 在新启用一个 buffer 时才 allocate node
  __STL_TRY {
    construct(finish.cur, t_copy);
    finish.set_node(finish.node + 1);
    finish.cur = finish.first;
  }
  __STL_UNWIND(deallocate_node(*(finish.node + 1)));
}

void deque<T, Alloc, BufSize>::reallocate_map(size_type nodes_to_add, bool add_at_front) {
  size_type old_num_nodes = finish.node - start.node + 1;
  size_type new_num_nodes = old_num_nodes + nodes_to_add;
  
  map_pointer new_nstart;
  if (map_size > 2 * new_num_nodes) {
    // 不需要申请内存空间
    new_nstart = map + (map_size - new_num_nodes) / 2 + (add_at_front ? nodes_to_add : 0);
    if (new_nstart < start.node) {
      // 如果当前的起始位置比原来要靠前，可以从 start 开始复制
      copy(start.node, finish.node + 1, new_nstart);
		} else {
      // 如果当前的起始位置比原来靠后，必须从 finish -> start 的方向进行复制才能保证数据完整性
      copy_backward(start.node, finish.node + 1, new_nstart);
    }
  } else {
    // 申请一块新的空间给 map 使用，该 map 数组比原来大一倍
    size_type new_map_size = map_size + max(map_size, nodes_to_add) + 2;
    map_pointer new_map = map_allocate::allocate(new_map_size);
    new_nstart = new_map + (new_map_size - new_num_nodes) / 2 + (add_at_front ? nodes_to_add : 0);
    copy(start.node, finish.node + 1, new_nstart);
    // 释放原 map
    map_allocator::deallocate(map, map_size);
    
    // 设置新 map
    map = new_map;
    map_size = new_map_size;
  }
  
  start.set_node(new_nstart);
  finish.set_node(new_nstart + old_num_nodes - 1);
}
```



#### 一些实现细节

1. 如何调整大小？

   当 map 数组中所有的元素都已经被使用，需要另一个块时，map 大小加倍，旧的指针将位于新的 map 数组中间。

   扩容只影响 map 数组，正在使用的块不会有任何改变。

2. 对于非开头/结尾的 insert/erase，将判断该位置是离哪里比较近：

   * 如果离 begin 更近，则将该位置之前的元素向后移动一位；
   * 如果离 end 更近，则将该位置之后的元素向前移动一位。

   这样的移动次数总比同规模的 vector 要少一半，平均时间复杂度仍为\\(O(n) \\) 。

### list

链表，顺序容器。其特征有：

* 访问或修改序列中的任意项的时间复杂度为 \\(O(n)\\)
* 给出序列某一位置的迭代器，在这个位置上插入或删除项所花费的时间复杂度为常数。

是一种环状双向链表。

![stl-container-list.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652928146888/I-2BTvila.png align="left")

与 vector 和 deque 相比，list 必须使用 iterator 来遍历，并且不支持随机遍历。

#### 常用方法接口

| 方法                             | 功能                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| list<double> x                   | x 称为一个空链表                                             |
| list<double> weights(x)          | list 对象 weights 包含了对 list 对象 x 的拷贝                |
| weights.push_front(8.3)          | 在 weights 开头插入 8.3                                      |
| weights.push_back(107.2)         | 在 weights 尾部插入 107.2                                    |
| weights.insert(itr, 125.0)       | 在 iter 所在的位置插入 125.0；<br />返回指向刚插入项的迭代器 |
| weights.pop_front()              | 删除 weights 开头的项                                        |
| weights.pop_back()               | 删除 weights 尾部的项                                        |
| weights.erase(iter)              | 删除 iter 所在位置的项，只有该删除位置上的迭代器和引用失效   |
| weights.erase(iter1, iter2)      | 删除 weights 中所在位置 [iter1, iter2) 之间的项              |
| weights.size()                   | 返回 weights 项数量                                          |
| weights.empty()                  | 空则返回 true，非空则返回 false                              |
| weights.splice(itr, old_weights) | 把 old_weights 中的所有项放在 weights 里 itr 所在位置的前面。<br />不论 weights 或 old_weights 里原先有多少项，这个方法的时间总是常数 |
| weights.sort()                   | 根据 operator < 排序 weights 中的项。                        |

#### 迭代器接口

list 类支持 bidirectional iterators，而不是随机访问迭代器。

```c++
template<class T, class Ref, class Ptr>
struct __list_iterator {
  typedef __list_iterator<T, T&, T*> iterator;
  typedef __list_iterator<T, Ref, Ptr> self;
  
  typedef bidirectional_iterator_tag iterator_category; // 双向的
  typedef T value_type;
  typedef Ptr pointer;
  typedef Ref reference;
  typedef __list_node<T>* link_type;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
  
  link_type node; // 普通指针，指向 list 节点
  
  // constructor
  __list_iterator(link_type x) : node(x) {}
  __list_iterator() {}
  __list_iterator(const iterator& x) : node(x.node) {}
  
  bool operator==(const self& x) const { return node == x.node; }
  bool operator!=(const self& x) const { return node != x.node; }
  reference operator*() const { return (*node).data; }
  reference operator->() const { return &(operator*()); }
  self& operator++() { // ++i;
    node = (link_type)((*node)next);
    return *this;
  }
  self operator++(int) { // i++;
    self tmp = *this;
    ++*this;
    return tmp;
  }
  self& operator--() { // --i;
    node = (link_type)((*node).prev);
    return *this;
  }
  self operator--(int) { // i--;
    self tmp = *this;
    --*this;
    return tmp;
  }
}
```

#### list 实现

```c++
template<class T>
class list {
  protected:
  	unsigned length;
  	struct list_node {
      list_node* next;
      list_node* prev;
      T data; // 保存一个项
    }; // list_node
  
  	list_node* node; // 头节点


  	// 内存管理所需要的成员
		list_node* next_avail; // 指向下一次插入将使用的节点
		list_node* last; // 指向缓冲区末尾的下一个节点
    struct list_node_buffer {
      list_node_buffer* next_buffer;
      list_node* buffer;
    };
    list_node_buffer* buffer_list; // 存储 buffer 的链表，头节点不存数据
  	list_node* free_list; // 存储被释放的头节点
}
```

在 insert 时，需要类似链表的那种处理，同时需要创建一个新的 list_node：

```c++
list_node *tmp = new list_node;
```

这个效率很低，每个 list_node 都有相同的大小，但是通用堆管理器不能利用这种一致性。依赖于计算机系统，每次调用 new 运算符都产生一个中断，而且这些重复的中断将大大降低项目运行速度。

可以令 list 类开发它自己的**内存管理**例程，`get_node` 分配链表节点，`put_node` 回收空间，因此，insert 的实现为：

```c++
iterator insert(iterator position, const T& x) {
  list_node* tmp = get_node(); // 从内存中获取一个 node
  construct(value_allocator.address((*tmp).data), x);
  (*tmp).next = position.node;
  (*tmp).prev = (*position.node).prev;
  (*((*position).node).prev).next = tmp;
  (*position.node).prev = tmp;
  ++length;
  return tmp;// position 没变，返回的是新 insert 的 iterator
}
```

如果当前的缓冲区已经满了，需要再申请一个相同大小的缓冲区放在 buffer_list 里面。

这些各种各样的缓冲区和链表空间不会被回收，除非清除链表。因此，如果应用创建一个很大的链表，然后删除几乎全部的项，所有的链表空间仍然被占用着。

## Adapter

### stack

底层 container 默认是 deque。先进后出的数据结构。

### queue

底层 container 默认是 deque，先进先出 （FIFO）的数据结构。queue 不允许有遍历行为。

### heap

complete binary tree （完全二叉树）适合作为 priority queue 的底层机制。

### priority_queue

具有权值观念的 queue，允许在底端加入元素，从顶端取出元素。

# 参考

《STL 源码解析》

[SGI STL 内存分配方式及malloc底层实现分析](https://www.cnblogs.com/LUO77/p/5824625.html)

[第二级配置器 free-list 详解](https://blog.csdn.net/u014587123/article/details/81628151)

