## 字符串分析
### 字符串结构体
php内核字符串定义在`Zend/zend_types.h`文件中
```
struct _zend_string {
	zend_refcounted_h gc;               /* 字符串类别及引用计数 */
	zend_ulong        h;                /* 字符串的哈希值 */
	size_t            len;              /* 字符串的长度 */
	char              val[1];           /* 柔性数组 */
};
```
`zend_refcounted_h`对应的结构体：
```
typedef struct _zend_refcounted_h {
	uint32_t         refcount;			/* reference counter 32-bit */
	union {
		struct {
			ZEND_ENDIAN_LOHI_3(
				zend_uchar    type,
				zend_uchar    flags,    /* used for strings & objects */
				uint16_t      gc_info)  /* keeps GC root number (or 0) and color */
		} v;
		uint32_t type_info;
	} u;
} zend_refcounted_h;
```



`Zend/zend_types.h`文件第87行定义别名
```
typedef struct _zend_string     zend_string;
```
`char val[1]`这里是C语言中的变长结构体的使用，并不是一个char的数组，`zend_string`在分配内存时会多分配一点，类似`malloc(sizeof(zend_string)+字符串长度)`，`val`正好是这个字符串开始地址，多出来的一个字节，存入空字符'\0'。


### 字符串初始化
使用函数`zend_string_init`初始化

```
static zend_always_inline zend_string *zend_string_init(const char *str, size_t len, int persistent)
{
	zend_string *ret = zend_string_alloc(len, persistent);

	memcpy(ZSTR_VAL(ret), str, len);
	ZSTR_VAL(ret)[len] = '\0';  // 最后一个字符，存入'\0'
	return ret;
}
```

继续看`zend_string_alloc`函数
```
static zend_always_inline zend_string *zend_string_alloc(size_t len, int persistent)
{
	zend_string *ret = (zend_string *)pemalloc(ZEND_MM_ALIGNED_SIZE(_ZSTR_STRUCT_SIZE(len)), persistent);

	GC_REFCOUNT(ret) = 1;
#if 1
	/* optimized single assignment */
	GC_TYPE_INFO(ret) = IS_STRING | ((persistent ? IS_STR_PERSISTENT : 0) << 8);
#else
	GC_TYPE(ret) = IS_STRING;
	GC_FLAGS(ret) = (persistent ? IS_STR_PERSISTENT : 0);
	GC_INFO(ret) = 0;
#endif
	zend_string_forget_hash_val(ret);
	ZSTR_LEN(ret) = len;
	return ret;
}
```
这里有一个`pemalloc`，它其实是一个宏（定义在`Zend/zend_alloc.h`中），这个宏根据persistent参数来决定是否分配持久内存。
```
#define pemalloc(size, persistent) ((persistent)?__zend_malloc(size):emalloc(size))
```
`__zend_malloc`其实就是调用了`malloc`函数（`Zend/zend_alloc.c`中）：
```
ZEND_API void * __zend_malloc(size_t len)
{
	void *tmp = malloc(len);
	if (EXPECTED(tmp || !len)) {
		return tmp;
	}
	zend_out_of_memory();
}
```



### 字符串扩展
使用函数`zend_string_extend`扩展
```
static zend_always_inline zend_string *zend_string_extend(zend_string *s, size_t len, int persistent)
{
	zend_string *ret;

	ZEND_ASSERT(len >= ZSTR_LEN(s));
	if (!ZSTR_IS_INTERNED(s)) {
		if (EXPECTED(GC_REFCOUNT(s) == 1)) {
			ret = (zend_string *)perealloc(s, ZEND_MM_ALIGNED_SIZE(_ZSTR_STRUCT_SIZE(len)), persistent);
			ZSTR_LEN(ret) = len;
			zend_string_forget_hash_val(ret);
			return ret;
		} else {
			GC_REFCOUNT(s)--;
		}
	}
	ret = zend_string_alloc(len, persistent);
	memcpy(ZSTR_VAL(ret), ZSTR_VAL(s), ZSTR_LEN(s) + 1);
	return ret;
}
```