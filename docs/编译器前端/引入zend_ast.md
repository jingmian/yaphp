# 引入zend_ast

细心的小伙伴会发现，在我们实现算数表达式的时候，直接在语法分析的过程（也就是调用`bison`的阶段）对表达式进行了求值。现在，我们来引入抽象语法树的概念。在`php-src`里面，对应的就是`zend_ast`结构了。

我们先来编写一下`zend_language_parser.y`文件，以让我们对整个过程有一个清晰的认识。

首先，我们需要定义一个`union`：

```bison
%union {
    zend_ast *ast;
}
```

这个是什么呢？实际上，这个是`YYSTYPE`，也就是`yylval`。而`yylval`我们会在词法分析的时候用到。如果我们不定义这个`union`的话，那么`YYSTYPE`默认是`int`类型。那么，我们在词法分析的时候，就只能够使用`yylval`去接收`int`类型的`token`了。但是，我们现在在词法分析的时候，会去为`T_LNUMBER`生成一个`zend_ast`，并且赋值给`yylval`，以便在语法分析的阶段，直接使用这个这个`zend_ast`。（实际上，我们也可以把这一步放到语法分析来做，但是，没有必要。）

我们需要对`token`定义做一些修改：

```bison
%token <ident> T_ECHO    "'echo'"
%token <ast> T_LNUMBER   "integer"
```

这里`%token <ast> T_LNUMBER`里面的`ast`就是说明了我们的`T_LNUMBER`这个`token`的值类型是`ast`。

`OK`，除了声明`token`的类型之外，我们还需要对文法里面的非终结符进行声明：

```bison
%type <ast> top_statement statement
%type <ast> expr
%type <ast> scalar
%type <ast> top_statement_list
```

现在，让我们看看我们的语法规则：

```bison
start:
    top_statement_list  { CG(ast) = $1; }
;

top_statement_list:
        top_statement_list top_statement { $$ = zend_ast_list_add($1, $2); }
    |   %empty { $$ = zend_ast_create_list(0, ZEND_AST_STMT_LIST); }
;

top_statement:
    statement                       { $$ = $1; }
;

statement:
    T_ECHO expr ';'     { $$ = $2; }
;

expr:
    expr '+' expr   { $$ = zend_ast_create_binary_op(ZEND_ADD, $1, $3); }
|   expr '-' expr   { $$ = zend_ast_create_binary_op(ZEND_SUB, $1, $3); }
|   expr '*' expr   { $$ = zend_ast_create_binary_op(ZEND_MUL, $1, $3); }
|   expr '/' expr   { $$ = zend_ast_create_binary_op(ZEND_DIV, $1, $3); }
|   '(' expr ')'    { $$ = $2; }
|   scalar { $$ = $1; }
;

scalar:
    T_LNUMBER   { $$ = $1; }
;
```

这里的规则和`php-src`是保持一致的，但是可能会有一点不同（一些目前没有必要的东西被我删了）。

可以看到，我们在最顶层，也即是非终结符`start`的那条规则里，它的动作是把`ast`赋值给`CG(ast)`。熟悉`PHP`内核的小伙伴应该是能够立马知道这个`CG(ast)`是做什么的。这个`CG(ast)`保存的是，在编译`PHP`代码阶段的所有`zend_ast`。

我们接着看：

```bison
top_statement_list:
        top_statement_list top_statement { $$ = zend_ast_list_add($1, $2); }
    |   %empty { $$ = zend_ast_create_list(0, ZEND_AST_STMT_LIST); }
;
```

它表达的含义一句话来总结就是，定义了一个`ZEND_AST_STMT_LIST`类型的`zend_ast`，然后这个`zend_ast`下面有多个子`zend_ast`，这些一个个的子`zend_ast`对应的就是我们`PHP`代码里的一条条语句（其实也不一定是一条条，例如`for`循环啥的，这种我们就不太好形容为条。反正大概就是一个语句的意思）。

```bison
statement:
    T_ECHO expr ';'     { $$ = $2; }
;
```

表示我们的语句包含`echo`语句。

```bison
expr:
    expr '+' expr   { $$ = zend_ast_create_binary_op(ZEND_ADD, $1, $3); }
|   expr '-' expr   { $$ = zend_ast_create_binary_op(ZEND_SUB, $1, $3); }
|   expr '*' expr   { $$ = zend_ast_create_binary_op(ZEND_MUL, $1, $3); }
|   expr '/' expr   { $$ = zend_ast_create_binary_op(ZEND_DIV, $1, $3); }
|   '(' expr ')'    { $$ = $2; }
|   scalar { $$ = $1; }
;

scalar:
    T_LNUMBER   { $$ = $1; }
;
```

这一段和我们前面的文章类似，但是，不同的地方在规则对应的动作，之前因为`token`的值都是`int`类型，所以，我们可以直接进行表达式的运算，但是，现在由于我们的`token`变成了`ast`类型，所以，我们就不能直接进行四则运算了（实际上，因为我们的语言是`C++`，所以我们可以对运算符进行重载，然而，这已经超出了我们教程的范围，所以我们就不去支持重载了，留给小伙伴们当作练习吧）。我们现在改为调用`zend_ast_create_binary_op`函数，而这个函数就是通过两个子`zend_ast`节点来生成一个四则运算的`zend_ast`，具体的函数实现我们后面会说。

改完了语法文件之后，我们就业需要改改词法文件：

```flex
[0-9]+ {
    int64_t lnum = atoi(yytext);
    yylval.ast = zend_ast_create_lnum(lnum);
    return T_LNUMBER;
}
```

我们上面说了，`yylval`是一个`union`类型的了，所以，我们得使用`yylval.ast`来接收`token`的值。而`zend_ast_create_lnum`函数就是用来把数字变成`zend_ast`的。

`OK`，我们现在可以来看具体的函数实现了。不过在这之前，我们需要先设计好我们的`zend_ast`结构，我们和`php-src`的保持一致：

```cpp
typedef uint16_t zend_ast_kind;
typedef uint16_t zend_ast_attr;

struct _zend_ast {
    zend_ast_kind kind;
    zend_ast_attr attr;
    zend_ast *child[1];
};

typedef struct _zend_ast_list {
    zend_ast_kind kind;
    zend_ast_attr attr;
    uint32_t children;
    zend_ast *child[1];
} zend_ast_list;

typedef struct _zend_ast_lnum {
    zend_ast_kind kind;
    zend_ast_attr attr;
    int64_t lnum;
} zend_ast_lnum;
```

前面的`_zend_ast`和`_zend_ast_list`是`php-src`里面的，小伙伴们可以在网上找到它们的区别。而`_zend_ast_lnum`是我自己引入的，表示这个`zend_ast`存了一个`lnum`的整数值。在`php-src`里面，这一块应该是`zend_ast_zval`，也就是存了一个`zval`。因为我们这篇文章还不想引入`zval`这个东西（因为我们的表达式都是整形值，所以没必要搞一个`zval`），所以我先简单处理了。

现在，让我们来看看函数实现了。

首先是`zend_ast_create_list`，我们在文件`Zend/zend_ast.cc`里面来进行定义：

```cpp
zend_ast *zend_ast_create_list(uint32_t init_children, zend_ast_kind kind) {
    zend_ast *ast;
    zend_ast_list *list;

    ast = (zend_ast *) malloc(zend_ast_list_size(4));
    list = (zend_ast_list *) ast;
    list->kind = kind;
    list->children = 0;

    return ast;
}
```

这个函数实现非常简单，首先，`malloc`出一块`zend_ast`的内存，然后，设置它的`kind`和`children`。其中`kind`对应这个`zend_ast`的类型，`children`表示这个`zend_ast`有一个子`zend_ast`。

然后是`zend_ast_list_add`函数：

```cpp
zend_ast *zend_ast_list_add(zend_ast *ast, zend_ast *op) {
    zend_ast_list *list = zend_ast_get_list(ast);
    list->child[list->children++] = op;
    return (zend_ast *) list;
}
```

这个函数就是设置`zend_ast_create_list`函数创建出来的`zend_ast`的`child`。设置一个`children`加一。所以，有几条语句，我们的这个`children`就是几了。

最后，是我们的`zend_ast_create_binary_op`函数：

```cpp
static inline zend_ast *zend_ast_create_binary_op(uint32_t opcode, zend_ast *op0, zend_ast *op1) {
    switch (opcode)
    {
    case ZEND_ADD:
        std::cout << "create + zend_ast" << std::endl;
        break;
    case ZEND_SUB:
        std::cout << "create - zend_ast" << std::endl;
        break;
    case ZEND_MUL:
        std::cout << "create * zend_ast" << std::endl;
        break;
    case ZEND_DIV:
        std::cout << "create / zend_ast" << std::endl;
        break;
    default:
        std::cout << "unknow operator" << std::endl;
        break;
    }
    return zend_ast_create_2(ZEND_AST_BINARY_OP, opcode, op0, op1);
}

zend_ast *zend_ast_create_2(zend_ast_kind kind, zend_ast_attr attr, zend_ast *child1, zend_ast *child2) {
    zend_ast *ast;

    ast = (zend_ast *) malloc(zend_ast_size(2));
    ast->kind = kind;
    ast->attr = attr;
    ast->child[0] = child1;
    ast->child[1] = child2;

    return ast;
}
```

这个`zend_ast_create_binary_op`实际上做的一件事情就是创建一个包含两个子`ast`的`zend_ast`，然后设置`zend_ast`的`kind`为`ZEND_AST_BINARY_OP`，并且设置对应的运算类型（即加、减、乘、除），最后设置运算符操作的两个子`ast`。

最后，我们编写一个打印`ast`的函数：

```cpp
void dump_ast(zend_ast *ast)
{
    if (ast->kind == ZEND_AST_LNUM) {
        zend_ast_lnum *ast_lnum = (zend_ast_lnum *) ast;
        std::cout << "kind: " << ast_lnum->kind << ", attr: " << ast_lnum->attr << ", value: " << ast_lnum->lnum << std::endl;
    } else {
        std::cout << "kind: " << ast->kind << ", attr: " << ast->attr << std::endl;
    }
}

void dump_compiler_globals()
{
    zend_ast *ast;
    std::deque<zend_ast *> queue;

    queue.push_back(CG(ast));

    while (!queue.empty())
    {
        ast = queue.front();

        if (ast->kind == ZEND_AST_STMT_LIST) {
            zend_ast_list *ast_list = (zend_ast_list *) ast;
            for (size_t i = 0; i < ast_list->children; i++)
            {
                queue.push_back(ast_list->child[i]);
            }
        } else if (ast->kind == ZEND_AST_BINARY_OP) {
            queue.push_back(ast->child[0]);
            queue.push_back(ast->child[1]);
        }
        dump_ast(ast);
        queue.pop_front();
    }

    return;
}
```

这个也很简单，实际上就是一个树的层次遍历。

`OK`，做完了这些工作之后，我们重新编译我们的`yaphp`（记得修改我们的`CMakeLists`）。

然后编写如下`yaphp`代码：

```php
echo 1 + 2 * 3;
```

输出结果如下：

```bash
create * zend_ast
create + zend_ast
kind: 129, attr: 0
kind: 515, attr: 1
kind: 65, attr: 0, value: 1
kind: 515, attr: 3
kind: 65, attr: 0, value: 2
kind: 65, attr: 0, value: 3
```

这个`ast`的输出大概如下图：

```txt
            stmt_list
        +
    1       *
        2       3
```

符合我们的预期。
