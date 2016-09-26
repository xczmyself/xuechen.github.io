##	1.内存操作（数据结构）

 - <font color=ff00ff size=4>cJSON结构体</font>

```c
typedef struct cJSON {
	struct cJSON *next,*prev;	//双向链表指针
	struct cJSON *child;		//指向第一个子节点的指针
	int type;					//value的类型

	char *valuestring;			//如果 value 是字符串，这为它的值
	int valueint;				//如果 value 是数字，这是整形值
	double valuedouble;			//如果 value 是数字，这是浮点型值
	char *string;				//如果是对象 key-value， key 值
} cJSON;
```

---
 
 - <font color=ff00ff size=4>cJSON\_malloc() & cJSON\_free() 函数</font>

函数使用了[HOOK技术](http://www.jianshu.com/p/150c284c0f3b)，简单地说，就是当对一些api功能不满意时，修改api函数功能。方法为通过 hook 接触到需要修改的 api 函数入口点，然后改变他的函数地址指向新的自定义函数。

```c
//头文件中定义了结构体与函数声明
typedef struct cJSON_Hooks {
      void *(*malloc_fn)(size_t sz);
      void (*free_fn)(void *ptr);
} cJSON_Hooks;
	//成员 malloc_fn 是一个函数指针（指向一个函数的指针）,有一个 size_t 类型参数，返回类型为 void *
	//成员 free_fn 也是一个函数指针，有一个 void * 类型参数，无返回值
extern void cJSON_InitHooks(cJSON_Hooks* hooks);


//主文件定义
static void *(*cJSON_malloc)(size_t sz) = malloc;
static void (*cJSON_free)(void *ptr) = free;
	//定义一个函数指针 cJSON_malloc 并初始化指向 malloc
	//定义一个函数指针 cJSON_free 并初始化指向 free，使用 cJSON_InitHooks 函数可以替换成用户自定义的 malloc free 函数
	
void cJSON_InitHooks(cJSON_Hooks* hooks)
{
    if (!hooks) { /* Reset hooks */
        cJSON_malloc = malloc;
        cJSON_free = free;
        return;
    }

	cJSON_malloc = (hooks->malloc_fn)?hooks->malloc_fn:malloc;
	cJSON_free	 = (hooks->free_fn)?hooks->free_fn:free;
		//这个条件运算符 + 函数指针 有点吊！对于 a ? b : c
		//实测，如果 a = NULL，判断为 FALSE，a != NULL 判断为 TRUE
}
```

示例：

```c
	//假如有一个自定义的 my_malloc my_free 函数
cJSON_Hooks *hooks = NULL;
hooks = (cJSON_Hooks *)calloc(1, sizeof(cJOSN_Hooks));
hooks->malloc_fn = my_malloc;
hooks->free_fn = my_free;
cJSON_InitHooks( hooks );
	//这样再使用 cJSON_malloc, cJSON_free 函数就变成了调用 my_malloc, my_free 函数

```

---

 - <font color=ff00ff size=4>cJSON_strcasecmp()函数</font>

参数s1，s2指向两个常量字符串<br>
s1 > s2, returns 正数; <br>s1 == s2 returns 0; <br>s1 < s2 returns 负数.
 
```c
static int cJSON_strcasecmp(const char *s1,const char *s2)
{
	if (!s1) return (s1==s2)?0:1;
	if (!s2) return 1;
	for(; tolower(*s1) == tolower(*s2); ++s1, ++s2)
		if(*s1 == 0)	return 0;	//遇到了 '\0'，字符串结束
	return tolower(*(const unsigned char *)s1) - tolower(*(const unsigned char *)s2);
	
}
```

- 善于使用条件运算符！ if( !s2 ) 执行时已经保证 s1 != NULL
- *(++s1) 得到 s1 的第二个字符,如果 s1 s2 一个字符不相等就跳出循环
- tolower( '\0' ) == 0, tolower( '0' ) == 48
- 跳出循环说明 s1 s2 遇到了不相等字符，转换成 unsigned char 之后相减
 
---

- <font color=ff00ff size=4>cJSON_strdup()函数</font>

将传入的字符串复制一份，并返回指向复制的字符串的指针。
 
```c
static char* cJSON_strdup(const char* str)
{
      size_t len;
      char* copy;

      len = strlen(str) + 1;
      		// strlen() 函数不计算 \0 ，所以要多 +1
      if (!(copy = (char*)cJSON_malloc(len))) return 0;
      		//如果 malloc 失败，copy 为 NULL， 返回 0
      memcpy(copy,str,len);
      		// memcpy() 函数会复制 len 个字符，不会因为 '\0' 而结束，
      		//所以把 str 内的 '\0' 一并复制给了 copy 。
      return copy;
}

```
 
---

- <font color=ff00ff size=4>cJSON_CreateIntArray()函数</font>

创建一个数组。

```c
cJSON *cJSON_CreateIntArray(const int *numbers,int count){
	int i;
	cJSON *n=0,*p=0,*a=cJSON_CreateArray();
	for(i=0; a && i<count; i++){
		n=cJSON_CreateNumber(numbers[i]);
		if(!i)		// i == 0 时表示此时是第一个子节点
			a->child=n;	// child 指针指向链表 a 的第一个子节点
		else 
			suffix_object(p,n);
		p=n;
	}
	return a;
}

cJSON *cJSON_CreateArray(void)
{
	cJSON *item=cJSON_New_Item();
	if(item)	//检测创建成功，返回值不是 NULL
		item->type=cJSON_Array;
	return item;
}

static void suffix_object(cJSON *prev,cJSON *item) 
{
	prev->next=item;
	item->prev=prev;
}	//双向链接起 prev <-> item 
```
 
- for循环第一次执行时，创建了首节点 a 跟第一个子节点 n，并且 a -> child = n
- for循环最后一行 p = n，保证此时 n 和 p 指针都指向新的子节点
- 之后的每一个循环过程中，n 为指向新的子节点的指针，suffix_object(p, n) 保证 p节点（链表中最新子节点）与 n 节点（新创建的子节点）双向链接起来 p <—> n
- 然后执行的 p = n 使 p 指针指向了当前最新的子节点，然后执行下一次循环
- <font color = blue>最终的结果就是：<br>
	a—>child, child 的 next 和 prev 指针链接之后的各个节点<br>
	a 本身的 next 和 prev 指针没有指向
	除了 a 节点之外的 child 指针没有指向</font>
 
---

- <font color=ff00ff size=4>cJSON\_AddItemToArray()、cJSON_AddItemToObject()函数</font>

添加儿子节点有两种情况，一种是给 Array 添加节点，一种是给 Object 添加节点。<br>
给 Object 添加节点只是多了一步设置 key 值的操作。

```c
void   cJSON_AddItemToArray(cJSON *array, cJSON *item)
{
	cJSON *c=array->child;		// c 为 array->child 指针
	if (!item) 		//如果新增项为 NULL （错误检测）
		return; 
	if (!c) {
		array->child=item;	//如果 c == NULL 说明 array 没有子节点，即 array 只有其本身一个节点
	} else {
		while (c && c->next) 	//如果 c c->next 都不为 NULL，说明此是 c 指向的是链表的中间一个节点
			c=c->next; 	//当 c != NULL， c->next == NULL 时，c 为最后一个节点
		suffix_object(c,item);	//此时把新的 item 节点双向链接到 c 节点之后，成为新的最后一个子节点
	}
}

void   cJSON_AddItemToObject(cJSON *object,const char *string,cJSON *item)	
{
	if (!item) 
		return; 
	if (item->string) 	//如果 item->string 有值，首先清空
		cJSON_free(item->string);
	item->string=cJSON_strdup(string);	//设置为新的值
	cJSON_AddItemToArray(object,item);	//添加节点
}

//同时为了简化操作，在头文件中还定义了一些宏来快速添加，例如：
#define cJSON_AddNumberToObject(object,name,n)	cJSON_AddItemToObject(object, name, cJSON_CreateNumber(n))

```

- 用到指针时，注意首先要检测指针有效性，即安全性。是否为 NULL。
- 判断时，对于一个指针，用 if( !ptr ) 代替 if( ptr != NULL ),更简洁。
 
---

- <font color=ff00ff size=4>cJSON\_DetachItemFromArray()、cJSON_DeleteItemFromArray()函数</font>

节点的删除函数。分为 delete（）跟 detach（）两种。

detach：把一个节点从双向链表中分离出来，但是并没有释放他的内存，然后返回这个分离的节点。这样我们可以做更多事情（添加到其他对象、释放内存等）。<br>
delete：直接把这个节点的内存释放掉。

```c
cJSON *cJSON_DetachItemFromArray(cJSON *array,int which)
{
	cJSON *c=array->child; //双向链表是在 array->child 里实现的
	while (c && which>0) 
		c=c->next,which--;	// c 指向要分离的那个节点
	if (!c) 
		return 0;		//如果节点为 NULL，返回
	if (c->prev) 
		c->prev->next=c->next;	
	if (c->next) 
		c->next->prev=c->prev;	//分离该节点，并将其前后节点链接起来
	if (c==array->child) 
		array->child=c->next;	//如果该节点是 child 节点，则把 child 指针指向下一个节点
	c->prev=c->next=0;	//清空 next prev 指针，方便调用cJSON_Delete函数
	return c;		//返回该节点指针
}

void   cJSON_DeleteItemFromArray(cJSON *array,int which)
{
	cJSON_Delete(cJSON_DetachItemFromArray(array,which));
}

void cJSON_Delete(cJSON *c)	//可删除单个节点或者整个cJSON链表
{
	cJSON *next;
	while (c)
	{
		next=c->next;
		if (!(c->type&cJSON_IsReference) && c->child) 
			cJSON_Delete(c->child);
		if (!(c->type&cJSON_IsReference) && c->valuestring)
			cJSON_free(c->valuestring);
		if (!(c->type&cJSON_StringIsConst) && c->string) 
			cJSON_free(c->string);
		cJSON_free(c);
		c=next;	//对于单个节点， next == NULL,所以只释放了这一个节点
	}
	//先删除子节点
	//如果有 valuestring 和 string 成员的节点，要释放对应内存
	//最后释放自己
}

cJSON *cJSON_DetachItemFromObject(cJSON *object,const char *string)
{
	int i=0;
	cJSON *c=object->child;
	while (c && cJSON_strcasecmp(c->string,string)) 
		i++,c=c->next;	//对于object，先找到与 key 值对应的节点索引 i
	//如果这个节点存在，再根据 i 调用 cJSON_DetachItemFromArray（）函数
	if (c) 
			return cJSON_DetachItemFromArray(object,i);
			//返回一个函数！
	return 0;
}

void   cJSON_DeleteItemFromObject(cJSON *object,const char *string)
{
	cJSON_Delete(cJSON_DetachItemFromObject(object,string));
}


```

---

- <font color=ff00ff size=4>cJSON\_GetArrayItem()、cJSON_GetObjectItem()函数</font>

查找节点函数。<br>
由于cJSON使用的是双向列表，所以使用暴力遍历方法。

```c
cJSON *cJSON_GetArrayItem(cJSON *array,int item)
{
	cJSON *c=array->child;  
	while (c && item>0) 
		item--,c=c->next; 
	return c;	//返回第 item 个节点的地址
}

cJSON *cJSON_GetObjectItem(cJSON *object,const char *string)
{
	cJSON *c=object->child; 
	while (c && cJSON_strcasecmp(c->string,string)) 
		c=c->next; 
	return c;
	//返回 key 值为 string 的节点地址。跳出循环的两种情况：
	//1，找到了匹配字符串，strcasecmp函数返回 0 ;2，c == NULL
	//所以，要么返回匹配字符串地址，要么返回 NULL
}


```

---

- <font color=ff00ff size=4>cJSON\_ReplaceItemInArray()、cJSON_ReplaceItemInObject()函数</font>

给定一个链表头结点，给定要将哪一个节点 which 替换成新节点 newitem
 
```c
void   cJSON_ReplaceItemInArray(cJSON *array,int which,cJSON *newitem)
{
	cJSON *c=array->child;
	while (c && which>0) 
		c=c->next,which--;	//定位到那个节点
	if (!c) 
		return;
	newitem->next=c->next;
	newitem->prev=c->prev;	//  <-- newitem -->
	if (newitem->next) 
		newitem->next->prev=newitem;	//	newitem <--
	if (c==array->child) 
		array->child=newitem; 
	else 
		newitem->prev->next=newitem;	//	--> newitem
	c->next=c->prev=0;
	cJSON_Delete(c);		//重置节点 c 指针并删除节点
}

void   cJSON_ReplaceItemInObject(cJSON *object,const char *string,cJSON *newitem)
{
	int i=0;
	cJSON *c=object->child;	//都是从 object->child 开始的。
	while(c && cJSON_strcasecmp(c->string,string))
		i++,c=c->next;	//找到 string（key值）对应的那个节点
	if(c){
		newitem->string=cJSON_strdup(string);
		cJSON_ReplaceItemInArray(object,i,newitem);
			//如果节点存在，调用 cJSON_ReplaceItemInArray 函数替换节点
	}
}
```
- 可以看出，不要重复造轮子。 ReplaceObject 函数就是在 ReplaceArray 函数基础上增添了一部分，然后调用了 ReplaceArray 函数。
- 即： 专用函数 = 专用函数修改 + 调用通用函数。 
 
---

- <font color=ff00ff size=4>create\_reference()函数</font>

- 当同时增加一个节点到两棵树中时，可能会有深拷贝和浅拷贝的问题。（比如要拷贝 item ）
	- 浅拷贝：创建新的指针 ptr 指向 item， ptr —> item
	- 深拷贝：开辟了新的内存 new_item 并复制 item 内容到新内存 new\_item，创建新的指针 ptr 指向新的内存空间， ptr —> new\_item
- cJSON 解决这个问题方法是，增加一个节点为 Reference 类型

```c
static cJSON *create_reference(cJSON *item)
{
	cJSON *ref=cJSON_New_Item();	//malloc新的内存节点
	if (!ref) 
		return 0;	//每次 malloc 后要记得检测是否成功！
	memcpy(ref,item,sizeof(cJSON));	//把原节点信息全部拷贝到新开辟的空间
	ref->string=0;	//	新节点 key 值清空
	ref->type|=cJSON_IsReference;	//新节点的类型为 item 原类型 + Reference 类型
	ref->next=ref->prev=0;	//清空新节点的指针引用
	return ref;
}

void	cJSON_AddItemReferenceToArray(cJSON *array, cJSON *item)
{
	cJSON_AddItemToArray(array,create_reference(item));
}

void	cJSON_AddItemReferenceToObject(cJSON *object,const char *string,cJSON *item)
{
	cJSON_AddItemToObject(object,string,create_reference(item));
}

```
 
- 可以看到，一个大函数可以分解成更基本的几个小函数 **嵌套** 或者 **并列** 的方式。
 
 
 
 
 <br><br>

##	2.JSON解析

- <font color=ff00ff size=4>cJSON_Parse()函数</font>

解析主函数。其实是个二次封装的函数。
解析器实际上是一个***自动机***，由一系列递归函数组成的状态转移。

```c
static const char *ep;
	// 静态全局变量，用于错误处理，如果出现错误，ep 保存的就是出现错误的位置
const char *cJSON_GetErrorPtr(void) {return ep;}
	//找到错误位置函数


cJSON *cJSON_Parse(const char *value)
{
	return cJSON_ParseWithOpts(value,0,0);
	// 默认设置， value 值为要解析的字符串
}

cJSON *cJSON_ParseWithOpts(const char *value,const char **return_parse_end,int require_null_terminated)
{
	const char *end=0;
	cJSON *c=cJSON_New_Item();	//创建新节点
	ep=0;	//全局变量 ep 设为 0
	if (!c) 
		return 0;       /* memory fail */

	end=parse_value(c,skip(value));	//核心函数
	if (!end){
		cJSON_Delete(c);
		return 0;
	}	/* parse failure. ep is set. */

	if (require_null_terminated){
		end=skip(end);
		if (*end){
			cJSON_Delete(c);
			ep=end;
			return 0;
		}
	}
	if (return_parse_end) 
		*return_parse_end=end;
	return c;
}

static const char *skip(const char *in)	// 跳过字符串开头的空格（回车、换行等）
{
	while (in && *in && (unsigned char)*in<=32) 
		in++; 
	// 首先检测 in 字符串指针不为 NULL
	// 然后检测 *in （第一个字符存在），字符串不为空
	// 最后检测 第一个字符是不是空格、换行等。（ ASCII 前32个字符为不可见字符）
	return in;
}
```

---

- <font color=ff00ff size=4>parse\_value()函数</font>

解析核心函数。参数 item 为创建的新节点，参数 value 为要解析的字符串。
<br>value 值有以下几种可能。
![](http://ww2.sinaimg.cn/mw690/64ce1b95jw1f85tnyqojyj20hc082aaq.jpg)

<br> parse_value 函数先检测 value 是否为空，是否是 true, false, null，如果是的话可以直接处理。

如果不是的话，通过比较判断value字符串的第一个字符来判断数据类型，并调用对应的解析函数来进行解析。

```c
static const char *parse_value(cJSON *item,const char *value)
{
	if (!value)						return 0;	/* Fail on null. */
	if (!strncmp(value,"null",4))	{ item->type=cJSON_NULL;  return value+4; }
	if (!strncmp(value,"false",5))	{ item->type=cJSON_False; return value+5; }
	if (!strncmp(value,"true",4))	{ item->type=cJSON_True; item->valueint=1;	return value+4; }
	//若 value 为 “true”， value + 4 后 value 指针指向 'e' 后面那个字符。（若 'e' 是最后一个字符，则指向 '\0'
	//指向 '\0' 时， if(!value) 为假，空字符串而已。
	//若 value == NULL， if(!value) 为真。
	//题外话，计算数组长度的时候，若有 char ptr[5]，有奇技淫巧 *(&ptr + 1) - ptr 为 5
	if (*value=='\"')				{ return parse_string(item,value); }
	if (*value=='-' || (*value>='0' && *value<='9'))	{ return parse_number(item,value); }
	if (*value=='[')				{ return parse_array(item,value); }
	if (*value=='{')				{ return parse_object(item,value); }

	ep=value;	//如果执行到这里，说明上面的判断全为假，置 ep 为当前位置，返回
	return 0;	/* failure. */
}
```

---

- <font color=ff00ff size=4>parse\_string()函数</font>

如果 parse_value 函数检测到第一个非空字符为 ' \" '，说明 value 为一个字符串。调用 parse\_string 函数进行解析。

string 字符串有下图所示可能。
![](http://ww3.sinaimg.cn/mw690/64ce1b95jw1f85xlskw85j20h80bvjs5.jpg)

```c
static const char *parse_string(cJSON *item,const char *str)
{
	const char *ptr=str+1;	// ptr 过滤掉 str 字符串的 “ 字符
	char *ptr2;
	char *out;
	int len=0;	//字符串长度
	unsigned uc,uc2;
	if (*str!='\"') {ep=str;return 0;}	//若str不以 " 开头，则不是字符串
	
	while (*ptr!='\"' && *ptr && ++len) 
		if (*ptr++ == '\\') 	
			ptr++;		/* Skip escaped quotes. */
		//计算字符串长度（直到遇到下一个' " '截止，跳过转义字符 '\'）
		//遇到转义字符时跳过转义字符后面的字符
	
	out=(char*)cJSON_malloc(len+1);	//多一个空间存放 '\0'
	if (!out) return 0;		//每次 malloc 后记得检查是否开辟成功
	
	ptr=str+1;ptr2=out;
	//ptr 重新指向非引号的第一个字符，新开辟的空间指针为 ptr2
	//下面开始进行从 ptr 到 ptr2 的拷贝
	while (*ptr!='\"' && *ptr)
	{
		if (*ptr!='\\') *ptr2++=*ptr++;	//不是转义字符直接拷贝
		else
		{
			ptr++;	//跳过转移字符，ptr 现在指向 \? 的 ？
			switch (*ptr)
			{
				case 'b': *ptr2++='\b';	break;
				case 'f': *ptr2++='\f';	break;
				case 'n': *ptr2++='\n';	break;
				case 'r': *ptr2++='\r';	break;
				case 't': *ptr2++='\t';	break;
				case 'u':	 // utf-16 转 utf-8 略过。
					uc=parse_hex4(ptr+1);ptr+=4;	/* get the unicode char. */

					if ((uc>=0xDC00 && uc<=0xDFFF) || uc==0)	break;	/* check for invalid.	*/

					if (uc>=0xD800 && uc<=0xDBFF)	/* UTF16 surrogate pairs.	*/
					{
						if (ptr[1]!='\\' || ptr[2]!='u')	break;	/* missing second-half of surrogate.	*/
						uc2=parse_hex4(ptr+3);ptr+=6;
						if (uc2<0xDC00 || uc2>0xDFFF)		break;	/* invalid second-half of surrogate.	*/
						uc=0x10000 + (((uc&0x3FF)<<10) | (uc2&0x3FF));
					}

					len=4;if (uc<0x80) len=1;else if (uc<0x800) len=2;else if (uc<0x10000) len=3; ptr2+=len;
					
					switch (len) {
						case 4: *--ptr2 =((uc | 0x80) & 0xBF); uc >>= 6;
						case 3: *--ptr2 =((uc | 0x80) & 0xBF); uc >>= 6;
						case 2: *--ptr2 =((uc | 0x80) & 0xBF); uc >>= 6;
						case 1: *--ptr2 =(uc | firstByteMark[len]);
					}
					ptr2+=len;
					break;
				default:  *ptr2++=*ptr; break;	//如果没有匹配，就把这个字符拷贝到 ptr2 中
			}
			ptr++;
		}
	}
	*ptr2=0;
	if (*ptr=='\"') ptr++;
		//返回的指针是指向 ' " ' 字符后面字符，也就是string结束后下一个字符
	item->valuestring=out;
	item->type=cJSON_String;
	return ptr;
}
```

---

- <font color=ff00ff size=4>parse\_number()函数</font>

如果 parse_value 函数检测到第一个非空字符为 负号 或者为 0~9 之间数字，说明 value 为一个数字。调用 parse\_number 函数进行解析。

number 数字有下图所示可能。
![](http://ww2.sinaimg.cn/mw690/64ce1b95jw1f85xlrhjuzj20ho07xaab.jpg)

```c
/* Parse the input text to generate a number, and populate the result into item. */
// 参数：一个 cJSON 节点 item，一个字符串（value） num
static const char *parse_number(cJSON *item,const char *num)
{
	double n=0,sign=1,scale=0;
	int subscale=0,signsubscale=1;

	if (*num=='-') sign=-1,num++;	/* Has sign? */
	if (*num=='0') num++;			/* is zero */
	if (*num>='1' && *num<='9')	
		do	
			n=(n*10.0)+(*num++ -'0');	
		while (*num>='0' && *num<='9');	//如果下一个字符还是数字就循环
		
	if (*num=='.' && num[1]>='0' && num[1]<='9'){
			// num[1] 为 '.' 后面的第一个字符（num[1]是 *++num,为了不改变指针
			//还能检测到小数点后面的第一个字符是不是数字，所以用了数组的方法！）
		num++;		
		do	
			n=(n*10.0)+(*num++ -'0'),scale--; 
			//依旧 *10 ，没写入小数，而是使用 scale 记录小数点位置
		while (*num>='0' && *num<='9');
	}	/* Fractional part? */
	
	if (*num=='e' || *num=='E'){		/* Exponent? */
		num++;
		if (*num=='+') num++;	
		else if (*num=='-') signsubscale=-1,num++;		/* With sign? */
		while (*num>='0' && *num<='9') 
			subscale=(subscale*10)+(*num++ - '0');
			// 10 的多少次方
	}
	// n 记录的纯数字，比如123.45e6，n 只记录了 12345
	// scale 为 -2，subscale 为 6，signsubscale 为 1

	n=sign*n*pow(10.0,(scale+subscale*signsubscale));
	// 计算最终结果时，小数（10的负几次方） 跟 e指数（10的几次方） 冲抵
	
	item->valuedouble=n;
	item->valueint=(int)n;
	item->type=cJSON_Number;
	// 解析后的值写入 item 对应位置
	return num;
	// 此时的 num 指针已经指向了整数字符串后的下一个字符，因为前面语句都
	// 是使用 num++ 再判断。判断失败跳出循环后，num 已经 ++ 了。
}
```
- 画流程图可以让思路清晰很多很多，别急着写代码，先思考应该怎样写。
- 处理数字时，不必着急算入小数点，可以在最后跟 e指数 一起计算，更有条理也更不容易出错。

---

- <font color=ff00ff size=4>parse\_array()函数</font>

数组形式是以 ' [ ' 开头，然后是节点内容，节点以 ' , ' 分隔符分隔，可能有多个节点(即多个 ' , ' )，分隔符前后可能有空格，最后以 ' ] ' 结尾。

数组内成员可能是 string，可能是 value，可能是 array，可能是 object。所以要递归嵌套使用 parse\_value() 函数。


```c
/* Build an array from input text. */
static const char *parse_array(cJSON *item,const char *value)
{
	cJSON *child;
	if (*value!='[')	{ep=value;return 0;}	/* not an array! */

	item->type=cJSON_Array;
	value=skip(value+1);	//跳过 '[' 字符后面的空格回车等
	if (*value==']') return value+1;	/* empty array. */
		//此时 value 已经在 skip() 中 +1 了，所以指向的是 '[' 后第一个
		//非空字符。 return value+1; 返回值为 ']' 后第一个字符('\0'?)

	item->child=child=cJSON_New_Item();	//malloc创建空间
	if (!item->child) return 0;		 //malloc错误检查
	value=skip(parse_value(child,skip(value)));
		//递归调用 parse\_value 函数，跳过空格符
	if (!value) return 0;

	//解析完一次之后， value 指针指向的是解析完成后的下一个字符 ！！！
	//所以可以直接判断该字符是不是分隔符 ','
	while (*value==',')	//新节点标志
	{
		cJSON *new_item;
		if (!(new_item=cJSON_New_Item())) return 0; 	/* memory fail */
		child->next=new_item;new_item->prev=child;child=new_item;
		// child 指向 new_item
		value=skip(parse_value(child,skip(value+1)));
		// skip(value+1) 跳过了 ',' 字符，递归调用 parse\_value 函数解析
		if (!value) return 0;	/* memory fail */
	}

	if (*value==']') return value+1;	/* end of array */
	ep=value;return 0;	/* malformed. */
}

```

- 因为 Array 数组内可能包含各种类型的节点，所以在 Array 数组内要递归的调用 parse\_value 函数递归的解析。
- ***注意到：***每次解析完成后，value 指针都是指向解析完的下一个字符！
	例如：<br>
	"abc"123<br>
	parse\_string()结束后，value指针指向的是 1

---

- <font color=ff00ff size=4>parse\_object()函数</font>

对象节点的形式为：<br>
以 ' { '开头，' } ' 结尾，中间是 key : value 形式的键值对。多个键值对之间以 ' , ' 分隔符分隔。

其中 key 值为 string 形式，所以可以直接用 parse\_string 解析。<br>
value 值可以为任意类型节点，所以要用 parse\_value 解析。<br>
遇到 ' , ' 分隔符后重新解析 key : value 键值对。

```c
/* Build an object from the text. */
static const char *parse_object(cJSON *item,const char *value)
{
	cJSON *child;
	if (*value!='{')	{ep=value;return 0;}	/* not an object! */
	
	item->type=cJSON_Object;
	value=skip(value+1);
	if (*value=='}') return value+1;	/* empty array. */
	
	item->child=child=cJSON_New_Item();
	if (!item->child) return 0;	// malloc 错误检查
	value=skip(parse_string(child,skip(value)));
		//解析 key 值， key 值为字符串 string 类型
	if (!value) return 0;
		//如果 key 值后没东西了，出错
	child->string=child->valuestring;child->valuestring=0;
		//parse_string 将解析的字符串放到了 valuestring 内，但是键值对的 key 值
		//应该存放在 string 字段内，valuestring 存放 key:value 的 value 值
	if (*value!=':') {ep=value;return 0;}	/* fail! */
		// key 后面如果不是 ' : ' 报错。
	value=skip(parse_value(child,skip(value+1)));
		//再解析 key:value 的 value 值
	if (!value) return 0;
	
	while (*value==',')	//如果是多个 key:value 键值对，以 ' , ' 分隔
	{
		cJSON *new_item;
		if (!(new_item=cJSON_New_Item()))	return 0; /* memory fail */
		child->next=new_item;new_item->prev=child;child=new_item;
		value=skip(parse_string(child,skip(value+1)));
		if (!value) return 0;
		child->string=child->valuestring;child->valuestring=0;
		if (*value!=':') {ep=value;return 0;}	/* fail! */
		value=skip(parse_value(child,skip(value+1)));
		if (!value) return 0;
	}
	
	if (*value=='}') return value+1;	/* end of array */
	ep=value;return 0;	/* malformed. */
}
```







