几种在堆上分配内存的方式

```c
void *malloc(size_t size);
void free(void *ptr);
void *calloc(size_t nmemb, size_t size);
void *realloc(void *ptr, size_t size);
void *reallocarray(void *ptr, size_t nmemb, size_t size);
```



### 1. malloc

```c
void *malloc(int n);//n为要分配的字节数
```
该函数必须配合memset初始化
```c
int *p;
p = (int*)malloc(sizeof(int));
memset(p, 0, sizeof(int));
```
### 2. calloc
```c
void *calloc(int n, int size);
//n为要分配的元素个数，size为元素所占的字节数。
```
该函数默认初始化都为0，不需要memset手动初始化，适合用于数组创建。
### 3. realloc

```c
void *realloc(void *p, int n);
//p是堆上一块内存的地址，也就是我们分配的
//n是我们将p指向的内存空间大小字节数设置为n，新起一块大小为n的内存，并且将数据给复制过去。
//但是如果p指向的内存空间大于或等于n，不变。
```

### 4. free

```c
void free(void *p);
//释放该指针所指向的内容，但是该指针还是存在的，其实他是在栈上的，只不过指向的空间是在堆上。
//所以我们为了不让她成为野指针，我们再释放p后，设p为NULL；
//free(p);
//p = NULL;
```

### 5. reallocarray

  The reallocarray() function changes the size of the memory block pointed to by ptr to be large enough for an array of  nmemb  elements,
  each of which is size bytes.  It is equivalent to the call

  `realloc(ptr, nmemb * size);`
  However, unlike that realloc() call, reallocarray() fails safely in the case where the multiplication would overflow.  If such an over‐flow occurs, reallocarray() returns NULL, sets errno to ENOMEM, and leaves the original block of memory unchanged.

### 为什么free不用传入malloc的内存的大小，却可以正确的释放呢？

  事实上我们malloc的时候，并不是严格按照我们指定的大小去申请的，而是有四个字节的头部存放，我们到底申请了多少内存，所以我们free的时候其实是去查看前面的四个字节，到底多大，再往后偏移的释放空间。

  实际上你申请0的空间，他也不一定就是给你0的大小。

  并且在cpp中，我们new[]一块空间，也不仅仅是new的空间，首部有4个字节存放，你new的对象个数，所以我们必须使用delete[]去释放，他会检查前面的4个字节，查看到底有多少个对象，再去释放。

​	TODO：详细捋一捋内存模型