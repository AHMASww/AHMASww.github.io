---
layout: post
title: "Antlr4 C++ Visitor方法"
date: 2023-06-14 18:05:00 +0800
categories: jekyll update
---

# Antlr4 C++ Visitor方法

## Expr.g4文件

```g
grammar Expr;

prog
    :   stat+
    ;

stat
    :   expr NEWLINE            # printExpr
    |   ID '=' expr NEWLINE     # assign
    |   NEWLINE                 # blank
    |   'clear()'               # clear
    ;

expr
    :   expr ('*'|'/') expr     # MulDiv
    |   expr ('+'|'-') expr     # AddSub
    |   INT                     # int
    |   ID                      # id
    |   '(' expr ')'            # parens
    ;

ID
    :   [a-zA-Z]+
    ;

INT
    :   [0-9]+
    ;

NEWLINE
    :   '\r'* '\n'
    ;

MUL
    :   '*'
    ;

DIV 
    :   '/'
    ;

ADD 
    :   '+'
    ;

SUB 
    :   '-'
    ;

WS
    :   [ \t] -> skip
    ;
```

`#`表示标签，放置在一个备选分支的右侧，标签可以是任意标识符，但不能与规则冲突。

## 生成对应的CPP文件

```shell
antlr4 -no-listener -visitor -Dlanguage=Cpp Expr.g4
```

生成如下对应文件
```
-rw-rw-r-- 1    76  6月 14 16:12 ExprBaseVisitor.cpp
-rw-rw-r-- 1  1436  6月 14 16:12 ExprBaseVisitor.h
-rw-rw-r-- 1   642  6月 14 16:11 Expr.g4
-rw-rw-r-- 1  1531  6月 14 16:12 Expr.interp
-rw-rw-r-- 1  5830  6月 14 16:12 ExprLexer.cpp
-rw-rw-r-- 1  1200  6月 14 16:12 ExprLexer.h
-rw-rw-r-- 1  2385  6月 14 16:12 ExprLexer.interp
-rw-rw-r-- 1   137  6月 14 16:12 ExprLexer.tokens
-rw-rw-r-- 1 19082  6月 14 16:12 ExprParser.cpp
-rw-rw-r-- 1  4662  6月 14 16:12 ExprParser.h
-rw-rw-r-- 1   137  6月 14 16:12 Expr.tokens
-rw-rw-r-- 1    72  6月 14 16:12 ExprVisitor.cpp
-rw-rw-r-- 1  1084  6月 14 16:12 ExprVisitor.h
```

注意`ExprBaseVisitor.h`中的内容，会产生对应分支的visitor方法。
```
...
virtual std::any visitPrintExpr(ExprParser::PrintExprContext *ctx) override {
    return visitChildren(ctx);
  }
...
```

## 复写需要的方法

当前g4文件是来实现一个类似计算器的功能，需要复写遇到不同分支时对应的处理。`MyVisitor`是自己的一个实际操作的类。
```c++
#pragma once


#include "ExprBaseVisitor.h"


class MyVisitor : public ExprBaseVisitor
{
public:
    /// @brief visit assign (ID '=' expr NEWLINE)
    /// @param ctx 
    /// @return value 
    virtual std::any visitAssign(ExprParser::AssignContext *ctx) override;

    /// @brief visit print expr (expr NEWLINE)
    /// @param ctx 
    /// @return 0
    virtual std::any visitPrintExpr(ExprParser::PrintExprContext *ctx) override;

    /// @brief visit INT (INT)
    /// @param ctx 
    /// @return value
    virtual std::any visitInt(ExprParser::IntContext *ctx) override;

    /// @brief visit ID (ID)
    /// @param ctx 
    /// @return value or 0
    virtual std::any visitId(ExprParser::IdContext *ctx) override;

    /// @brief visit MulDiv (expr ('*'|'/') expr)
    /// @param ctx 
    /// @return value 
    virtual std::any visitMulDiv(ExprParser::MulDivContext *ctx) override;

    /// @brief visit AddSub (expr ('+'|'-') expr)
    /// @param ctx 
    /// @return value 
    virtual std::any visitAddSub(ExprParser::AddSubContext *ctx) override;

    /// @brief visit parens ('(' expr ')') 
    /// @param ctx 
    /// @return 
    virtual std::any visitParens(ExprParser::ParensContext *ctx) override;

    /// @brief visit clear (clear())
    /// @param ctx 
    /// @return 
    virtual std::any visitClear(ExprParser::ClearContext *ctx) override;

private:
    std::map<std::string, int> memory;

};
```

```c++
#include "MyVisitor.h"


std::any MyVisitor::visitAssign(ExprParser::AssignContext *ctx)
{
    std::string id = ctx->ID()->getText();
    int value = std::any_cast<int>(visit(ctx->expr()));

    memory[id] = value;

    return value;
}

std::any MyVisitor::visitPrintExpr(ExprParser::PrintExprContext *ctx)
{
    int value = std::any_cast<int>(visit(ctx->expr()));
    std::cout << value << std::endl;

    return 0;
}

std::any MyVisitor::visitInt(ExprParser::IntContext *ctx)
{
    return std::stoi(ctx->INT()->getText());
}

std::any MyVisitor::visitId(ExprParser::IdContext *ctx)
{
    std::string id = ctx->ID()->getText();
    if (memory.find(id) != memory.end())
        return memory.find(id)->second;
    else
        return 0;
}

std::any MyVisitor::visitMulDiv(ExprParser::MulDivContext *ctx)
{
    int left = std::any_cast<int>(visit(ctx->expr(0)));
    int right = std::any_cast<int>(visit(ctx->expr(1)));

    if (ctx->MUL() != nullptr)
        return left * right;
    else
        return left / right;
}

std::any MyVisitor::visitAddSub(ExprParser::AddSubContext *ctx)
{
    int left = std::any_cast<int>(visit(ctx->expr(0)));
    int right = std::any_cast<int>(visit(ctx->expr(1)));

    if (ctx->ADD() != nullptr)
        return left + right;
    else
        return left - right;
}

std::any MyVisitor::visitParens(ExprParser::ParensContext *ctx)
{
    return visit(ctx->expr());
}

std::any MyVisitor::visitClear(ExprParser::ClearContext *ctx)
{
    memory.clear();
    std::cout << "clear memory, size : " << memory.size() << std::endl;
    
    return 0;
}
```

## main文件

main文件用来读取文件当中的表达式并用antlr来识别并计算。
```c++
#include <iostream>
#include <fstream>
#include <sstream>

#include "ExprLexer.h"
#include "ExprParser.h"
#include "MyVisitor.h"


bool ReadFile(std::string _filePath, std::string& _context)
{
    if (_filePath.empty())
        return false;

    std::ifstream in;
    
    in.open(_filePath, std::ios::in);
    if (in.is_open() == false)
    {
        return false;
    }
    else
    {
        std::stringstream buffer;
        buffer << in.rdbuf();
        _context = buffer.str();
    }

    return true;
}


int main(int argc, char** argv)
{
    if (argc != 2)
    {
        std::cout << "Usage : calc file" << std::endl;
        return -1;
    }
    
    std::string filePath = argv[1];
    std::string context;
    if (ReadFile(filePath, context) == false)
        return -1;
    std::cout << context << std::endl;

    antlr4::ANTLRInputStream input(context);
    ExprLexer lexer(&input);
    antlr4::CommonTokenStream tokens(&lexer);
    ExprParser parser(&tokens);

    antlr4::tree::ParseTree* tree = parser.prog();
    MyVisitor visitor;
    visitor.visit(tree);

    return 0;
}
```

### 编译

CMakeLists文件
```CMake
cmake_minimum_required(VERSION 3.10)

project(calc)

set(CMAKE_CXX_STANDARD 17)

aux_source_directory(. DIR_SRCS)

add_executable(calc ${DIR_SRCS})

target_link_libraries(calc antlr4-runtime)
```

### 表达式文件

```
193
a = 5
b = 6
a+b*2
(1+2)*3
clear()
```

### 输出

```
193
a = 5
b = 6
a+b*2
(1+2)*3
clear()

193
17
9
clear memory, size : 0
```
