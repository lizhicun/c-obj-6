# 第六章 执行期语义学
主要想讲对象在执行期间的什么时候开始构造，什么时候析构。

## 全局对象
如果有以下程序片段：
```
Matrix identity;

int main() {
  // identity 必须在此处被初始化
  Matrix m1 = identity;
  ...
  return 0;
}
```
* identity的构造时间：在 main() 函数中第一次用到 identity 之前必须把 identity 构造出来。　　
* identity的析构时间：在 main() 函数结束之前把 identity 摧毁掉。　　
* 像 identity 这样的 global object，如果有 constructor 和 destructor，必须要静态的初始化和释放操作。  
* C++ 程序中所有的 global object 都放在 data segment 中，如果不给初值，那么所配置的内存内容为 0。虽然 class object 在编译时期可以放在 data segment 中且内容为 0，但 constructor 会在程序激活时才会调用。

## 局部静态对象
```
const Matrix& identity {
  static Matrix mat_identity;
  // ...
  return mat_identity;
}
```
Local static class object 保证以下语义：
* mat_identity 的 constructor 必须只能执行一次，虽然上述函数可能会被调用多次。
* mat_identity 的 destructor 必须只能执行一次，虽然上述函数可能会被调用多次。

cfront 的做法是引入一个临时性对象以保护 mat_identity 的初始化操作，第一次处理 identity() 时，这个临时对象被评估为 false，于是 constructor 会被调用，然后临时对象被改为 true。而另一端的 destructor 也一样。
```
// 被产生出来的临时对象，作为戒护只用
static struct Matrix *__0__F3 = 0;

// identity() 的名称会被 mangled
struct Matrix* identity_Fv() {
  static struct Matrix __1mat_identity;
  __0__F3 
    ? 0
    :(__ct__1MatrixFv ( & __1mat_identity ),
     (__0__F3 = (&__1mat_identity)));
  ...
}
```

### 对象数组
如果 Point 没有定义 constructor 和 destructor，那么只需配置足够的内存即可，不需要做其他事
```
Point knots[10];
```
如果 Point 明确定义了 default constructor，那么 这个 constructor 必须轮流施行于每个元素之上。在 cfront 中，使用了一个名为 vec_new() 的函数产生出以 class object 构造而成的数组。函数类型通常如下。
```
void* vec_new (
  void *array,      // address of start of array
  size_t elem_size, // size of each class object
  int elem_count,   // number of elements in array
  void (*constructor)( void* ),
  void (*destructor)( void*, char )
)
```
 constructor 和 destructor 参数是这个 class 的 default constructor 和 default destructor 的函数指针。
 
 ## new 和 delete 运算符
运算符 new 是由以下两步完成的  
1.通过适当的 new 运算符函数实体，配置所需内存
```
// 调用函数库中的 new 运算符
int *pi = __new(sizeof(int));
```
2.给配置得来的对象设立初值
```
*pi = 5;
```
其中，初始化操作应该在内存配置成功后才执行
```
// new 运算符的两个分离步骤
// given: int *pi = new int(5);
int *pi;
if (pi = __new(sizeof(int)))
  *pi = 5;
```
delete类似。

### 针对数组的 new 语义
```
int *p_array = new int[5];
```
vec_new() 不会被调用，因为 vec_new() 的主要功能是把 default constructor 施行于 class object 所组成的数组的每个元素上。　　
C++ 2.0 之前，程序员需要将数组的真正大小提供给 delete 运算符。所以删除一个数组应该这样：  
```
delete [array_size] p_array;
```
对于 delete 操作，其行为并不一定符合程序员的预期，比如对于这样一个 class
```
class Point {
 public:
  Point();
  virtual ~Point();
  // ...
};
class Point3d : public Point {
 public:
  Point3d();
  ~Point3d();
  // ...
}
```
如果我们这样配置
```
Point *ptr = new Point3d[10];
```
