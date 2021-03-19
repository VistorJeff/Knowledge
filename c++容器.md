# C++ STL容器总结

### 一、序列式容器

#### 1. vector

特点：

1. 一个动态分配的数组（当数组空间内存不足时，都会执行：**分配新空间-复制元素-释放原空间**）；
2. 当删除元素时，不会释放限制的空间，所以vector容器的容量（capacity）大于vector容器的大小（size）；
3. 对于删除或插入操作，执行效率不高，越靠后插入或删除执行效率越高；
4. 高效的随机访问容器；

创建vector对象：

```c++
// 创建一个空的vector
vector<int> v0;

// 创建一个拥有3个元素的vector，默认值是0
vector<int> v1(3);

// 创建一个拥有5个值为2的元素的vector
vector<int> v2(5, 2);

vector<int> v3(3, 1, v2.get_allocator());

vector<int> v4(v2);

vector<int> v5(5);
for (auto i : v5) {
    v5[i] = i;
}

vector<int> v6(v5.begin() + 1, v5.begin() + 3);

vector<int> v7(move(v2));

vector<int> v8{ {1, 2, 3, 4} };
```

