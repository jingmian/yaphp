# 准备工作

在写代码之前，我们很有必要先把编译`C++`代码的工作做好。主要涉及到以下几个方面：

```txt
1. 编写CMakeLists
2. 编写一个编译的脚本
```

## 编写CMakeLists

因为`CMakeLists.txt`的内容比较简单，所以我直接贴出我们的`CMakeLists.txt`文件的内容：

```cmake
cmake_minimum_required(VERSION 3.4)
project(yaphp)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_COMPILER clang++)

find_package(FLEX REQUIRED)
set(FlexOutput ${CMAKE_SOURCE_DIR}/src/Zend/zend_language_scanner.cc)
if(FLEX_FOUND)
    add_custom_command(
      OUTPUT ${FlexOutput}
      COMMAND ${FLEX_EXECUTABLE}
              --outfile=${FlexOutput}
              ${CMAKE_SOURCE_DIR}/src/Zend/zend_language_scanner.l
      COMMENT "Generating zend_language_scanner.cc"
    )
endif()

find_package(BISON REQUIRED)
set(BisonOutput ${CMAKE_SOURCE_DIR}/src/Zend/zend_language_parser.cc)
if(BISON_FOUND)
    add_custom_command(
      OUTPUT ${BisonOutput}
      COMMAND ${BISON_EXECUTABLE}
              --defines=${CMAKE_SOURCE_DIR}/src/Zend/zend_language_parser.h
              --output=${BisonOutput}
              ${CMAKE_SOURCE_DIR}/src/Zend/zend_language_parser.y
      COMMENT "Generating zend_language_parser.cc"
    )
endif()

add_executable(yaphp
    ${FlexOutput}
    ${BisonOutput}
)
include_directories(BEFORE ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_options(yaphp PUBLIC ${CMAKE_CXX_FLAGS} -Wall -Wno-deprecated-register -O0 -g)

message(STATUS "summary of build options:
    Install prefix:  ${CMAKE_INSTALL_PREFIX}
    Target system:   ${CMAKE_SYSTEM_NAME}
    Compiler:
      CXX compiler:    ${CMAKE_CXX_COMPILER}
")
```

我们来讲一下核心的东西，其他不清楚的地方，可以网上搜一下。

首先来看这段代码：

```cmake
project(yaphp)
```

我们把我们的这个项目叫做`yaphp`，即表示`Yet another php`。

然后是这段代码：

```cmake
find_package(FLEX REQUIRED)
set(FlexOutput ${CMAKE_SOURCE_DIR}/src/Zend/zend_language_scanner.cc)
if(FLEX_FOUND)
    add_custom_command(
      OUTPUT ${FlexOutput}
      COMMAND ${FLEX_EXECUTABLE}
              --outfile=${FlexOutput}
              ${CMAKE_SOURCE_DIR}/src/Zend/zend_language_scanner.l
      COMMENT "Generating zend_language_scanner.cc"
    )
endif()
```

这段代码做的事情是，通过`flex`，让`zend_language_scanner.l`文件生成`zend_language_scanner.cc`文件（如果不清楚`zend_language_scanner.l`的小伙伴不用着急，我们后面会讲）。并且，我们可以看到，我们把`zend_language_scanner.l`文件放在了`Zend`目录下，这实际上是和`php-src`（即`php`解释器）的一致的。

然后是这段代码：

```cmake
find_package(BISON REQUIRED)
set(BisonOutput ${CMAKE_SOURCE_DIR}/src/Zend/zend_language_parser.cc)
if(BISON_FOUND)
    add_custom_command(
      OUTPUT ${BisonOutput}
      COMMAND ${BISON_EXECUTABLE}
              --defines=${CMAKE_SOURCE_DIR}/src/Zend/zend_language_parser.h
              --output=${BisonOutput}
              ${CMAKE_SOURCE_DIR}/src/Zend/zend_language_parser.y
      COMMENT "Generating zend_language_parser.cc"
    )
endif()
```

这段代码做的事情是，通过`bison`，让`zend_language_parser.y`文件生成`zend_language_parser.cc`文件和`zend_language_parser.h`文件（如果不清楚`zend_language_parser.y`的小伙伴不用着急，我们后面会讲）。

然后是这段代码：

```cmake
add_executable(yaphp
    ${FlexOutput}
    ${BisonOutput}
)
```

表示，我们要把`${FlexOutput}`和`${BisonOutput}`编译成`yaphp`可执行文件。

`OK`，按照这个`CMakeLists.txt`的意思，我们来创建对应的文件。首先是文件`src/Zend/zend_language_scanner.l`：

```flex
%option noyywrap
%option nounput
%option noinput
%{
#include "zend_language_parser.h"
%}

%%
echo {
    return T_ECHO;
}

[;(),+*/-] return *yytext;

[0-9]+ {
    yylval = atoi(yytext);
    return T_NUMBER;
}

[\t\n ]+  /* ignore \t, \n, whitespace */

%%
```

然后是文件`src/Zend/zend_language_parser.y`：

```bison
%{
#include <stdio.h>
#include <string.h>

extern int yylex(void);
extern int yyparse(void);
extern FILE *yyin;
extern int yylineno;

int yywrap()
{
    return 1;
}

void yyerror(const char *s)
{
    printf("[error] %s, in line %d\n", s, yylineno);
}

int main(int argc, char const *argv[])
{
    const char *file = argv[1];
    FILE *fp = fopen(file, "r");

    if(fp == nullptr)
    {
        printf("cannot open %s\n", file);
        return -1;
    }

    yyin = fp;
    yyparse();

    return 0;
}
%}

%token T_ECHO T_NUMBER

%%

statement: %empty
|   T_ECHO expr    { printf("%d\n", $2); }
;

expr: %empty
|   T_NUMBER                      {$$ = $1;}
;

%%
```

然后，我们创建文件`tests/test1.php`：

```php
echo 1
```

## 编写编译的脚本

我们创建文件`rebuild.sh`：

```bash
#!/bin/bash
__DIR__=$(cd "$(dirname "$0")" || exit 1; pwd); [ -z "${__DIR__}" ] && exit 1

cd "${__DIR__}" && ./clean.sh && mkdir -p build && cd build && cmake .. && make
```

这段代码很简单，就是先调用`clean.sh`脚本做一些清理工作，然后调用`cmake`来生成`Makefile`，然后调用`make`来编译代码，生成`yaphp`。

然后创建文件`tools/cleaner.sh`：

```bash
#!/bin/bash
__DIR__=$(cd "$(dirname "$0")" || exit 1; pwd); [ -z "${__DIR__}" ] && exit 1

error(){ echo "[ERROR] $1"; exit 1; }
success(){ echo "[SUCCESS] $1"; exit 0; }
info(){ echo "[INFO] $1";}

workdir="$1"

if ! cd "${workdir}"; then
  error "Cd to ${workdir} failed"
fi

info "Scanning dir \"${workdir}\" ..."

if [ ! -f "./Makefile" ] && [ ! -f "./CMakeLists.txt" ]; then
  error "Non-project dir ${workdir}"
fi

info "CMake build dir will be removed:"

rm -rf -v ./build

info "Following files will be removed:"

find ${workdir}/src/Zend -name zend_language_scanner.cc -print0 | xargs -0 rm -f -v
find ${workdir}/src/Zend -name zend_language_parser.h -print0 | xargs -0 rm -f -v
find ${workdir}/src/Zend -name zend_language_parser.cc -print0 | xargs -0 rm -f -v

success "Clean '${workdir}' done"
```

这个脚本会清理掉`cmake`生成的一系列文件。

然后创建文件`clean.sh`：

```bash
#!/bin/bash
__DIR__=$(cd "$(dirname "$0")" || exit 1; pwd); [ -z "${__DIR__}" ] && exit 1

"${__DIR__}"/tools/cleaner.sh "${__DIR__}"
```

`OK`，现在，所以的事情都做好了，我们只需要执行脚本`rebuild.sh`：

```bash
./rebuild.sh

# 省略其他的输出
[100%] Linking CXX executable yaphp
[100%] Built target yaphp
```

现在，你将会在目录`build`下面看到编译好的`yaphp`。并且，细心的话，你会发现，在目录`src/Zend`下面，生成了文件`zend_language_scanner.cc`、`zend_language_parser.h`、`zend_language_parser.cc`。

现在，让我们执行这个`yaphp`：

```bash
./build/yaphp tests/test1.php
1
```

我们将会看到，`1`被打印了出来。
