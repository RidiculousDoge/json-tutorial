# leptjson的API接口

## Introduction：

- leptjson是轻量级的json-parser，参照https://github.com/miloyip/json-tutorial，用c语言实现。
- 本文档主要梳理其API接口，实现思路与API目标。

## OverView

主要的实现思路：利用主函数`lept_parse(lept_value* v,const char* json)`，将接收到的字符指针json解析到lept_value* 类型的v中。

- 定义的类型（结构体）：

  - `lept_value`

    ```c
    struct lept_value {
        union {
            struct { lept_value* e; size_t size; }a;    /* array:  elements, element count */
            struct { char* s; size_t len; }s;           /* string: null-terminated string, string length */
            double n;                                   /* number */
        }u;
        lept_type type;
    };
    ```

    u中保存具体的类型（字符串/浮点数/数组），`lept_type`保存类型。

  - `lept_type`：利用枚举类型自定义的json类型。总共有`NULL`,`True`,`False`,`Number`,`String`,`Array`,`Object`7种类型。

    ```c
    typedef enum { LEPT_NULL, LEPT_FALSE, LEPT_TRUE,\\
        LEPT_NUMBER, LEPT_STRING, LEPT_ARRAY, \\
        LEPT_OBJECT } lept_type;
    ```

  - `lept_context`：包含json文本内容和一个栈，包括栈的大小和栈顶。

    ```c
    typedef struct {
        const char* json;
        char* stack;
        size_t size, top;
    }lept_context;
    ```

  - 状态码定义：对于各类异常状态分配状态码。

    ```c
    enum {
        LEPT_PARSE_OK = 0,
        LEPT_PARSE_EXPECT_VALUE,
        LEPT_PARSE_INVALID_VALUE,
        LEPT_PARSE_ROOT_NOT_SINGULAR,
        LEPT_PARSE_NUMBER_TOO_BIG,
        LEPT_PARSE_MISS_QUOTATION_MARK,
        LEPT_PARSE_INVALID_STRING_ESCAPE,
        LEPT_PARSE_INVALID_STRING_CHAR,
        LEPT_PARSE_INVALID_UNICODE_HEX,
        LEPT_PARSE_INVALID_UNICODE_SURROGATE,
        LEPT_PARSE_MISS_COMMA_OR_SQUARE_BRACKET
    };
    ```

- parse函数定义

  - 主入口`lept_parse(lept_value* v,const char* json)`

    按照`json=ws [value] ws`的语法进行解析。

  - `lept_parse_value(lept_context* c,lept_value* v)`按照存储在c->json中的文本数据解析c，存储在lept_value*类型的v中。

    - 根据c->json的第一个字符进行类型的判断，然后进行具体的解析。

  - `lept_parse_literal(lept_context* c,lept_value* v)`：解析true，false和null

  - `lept_parse_number(lept_context* c,lept_value* v)`：解析浮点数类型。

    json的数字定义如下

    ```
    number = [ "-" ] int [ frac ] [ exp ]
    int = "0" / digit1-9 *digit
    frac = "." 1*digit
    exp = ("e" / "E") ["-" / "+"] 1*digit
    ```

    利用宏定义的`do {…} while(0)`判断digit和digit1-9；

  - `lept_parse_string(c,v)`：解析字符串：由于长度可变，采用栈的结构进行解析，解析开始时记录栈顶。解析完后记录长度并全部出栈。

    使用`EXPECT(c,‘\"’)进入字符串的解析。

    - 对于普通字符ch，只要ch>0x20就可以正常解析。直接入栈。
    - 对于转义字符`‘\t’,‘\b’,‘\n’,‘\r’,‘\f’`，直接入栈。
    - 对于utf-8格式的字符`‘\uxxxx'`，在tutorial04中有详细介绍。
    - 对于在字符串解析过程中遇到`‘\0’`，意味着字符串未闭合，返回错误码。

  - `lept_parse_array(c,v)`：