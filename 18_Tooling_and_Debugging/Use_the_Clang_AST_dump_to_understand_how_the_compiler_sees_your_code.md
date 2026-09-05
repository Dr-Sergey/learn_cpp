# Use the Clang AST dump to understand how the compiler sees your code

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/IntroductionToTheClangAST.html>  

---

## Topic Overview

When C++ code behaves in a surprising way - an implicit conversion you didn't intend, an overload that picked the wrong function, a `auto` type that deduced to something unexpected - the compiler's internal view of your code is what actually matters, not what you think you wrote. The Clang AST dump gives you exactly that internal view.

The AST (Abstract Syntax Tree) is how Clang represents your source code after parsing. Every expression, every declaration, every implicit conversion the compiler inserts is a node in this tree. Dumping it lets you see the compiler's interpretation of your code in full detail, including all the invisible things happening behind the scenes.

```cpp
Source code:              AST:
int x = 1 + 2;           VarDecl 'x' 'int'
                            ImplicitCastExpr <IntegralCast>
                              BinaryOperator '+'
                                IntegerLiteral 1
                                IntegerLiteral 2
```

Even this trivial example shows something interesting: there's an `ImplicitCastExpr` node that doesn't appear in source. The compiler inserted it. For complex code with user-defined conversions, templates, and overloaded operators, these implicit nodes can be numerous and surprising.

---

## Self-Assessment

### Q1: Run `-Xclang -ast-dump` and navigate the tree

Here's a small program. When you dump the AST, you'll see that the compiler generated an implicit copy constructor for `Widget`, identified the copy construction of `auto copy = w`, and recorded every constructor initializer in detail.

```cpp
// ast_demo.cpp
#include <string>

struct Widget {
    int id;
    std::string name;

    Widget(int i, std::string n) : id(i), name(std::move(n)) {}
};

int main() {
    Widget w(42, "hello");
    auto copy = w;
    return copy.id;
}
```

The `-fsyntax-only` flag tells Clang to parse and type-check without producing an object file. Combined with `-ast-dump`, this is the fastest way to inspect the tree.

```bash
# Dump the full AST:
$ clang++ -std=c++20 -Xclang -ast-dump -fsyntax-only ast_demo.cpp

# Key nodes in output:
# TranslationUnitDecl
# |-CXXRecordDecl 'Widget'
# | |-FieldDecl 'id' 'int'
# | |-FieldDecl 'name' 'std::string'
# | |-CXXConstructorDecl Widget 'void (int, std::string)'
# | | |-ParmVarDecl 'i' 'int'
# | | |-ParmVarDecl 'n' 'std::string'
# | | `-CompoundStmt
# | |   |-CXXCtorInitializer 'id'
# | |   |  `-ImplicitCastExpr
# | |   |    `-DeclRefExpr 'i'
# | |   `-CXXCtorInitializer 'name'
# | |     `-CallExpr 'std::move'
# | |       `-DeclRefExpr 'n'
# | |-CXXConstructorDecl implicit copy Widget
# | `-CXXConstructorDecl implicit move Widget
# |
# `-FunctionDecl 'main'
#   `-CompoundStmt
#     |-DeclStmt
#     | `-VarDecl 'w' 'Widget'
#     |   `-CXXConstructExpr 'Widget(int, std::string)'
#     |-DeclStmt
#     | `-VarDecl 'copy' 'Widget'  <-- 'auto' deduced to Widget
#     |   `-CXXConstructExpr 'Widget(const Widget&)'  <-- copy ctor
#     `-ReturnStmt
#       `-ImplicitCastExpr
#         `-MemberExpr '.id'
#           `-DeclRefExpr 'copy'

# Filter to specific function:
$ clang++ -std=c++20 -Xclang -ast-dump \
    -Xclang -ast-dump-filter=main -fsyntax-only ast_demo.cpp

# Colored output:
$ clang++ -std=c++20 -Xclang -ast-dump -fcolor-diagnostics \
    -fsyntax-only ast_demo.cpp
```

Notice that the AST shows `CXXConstructorDecl implicit copy Widget` - the compiler synthesized a copy constructor, and you can see it plainly. The `auto copy = w` line resolves to `VarDecl 'copy' 'Widget'` with a `CXXConstructExpr 'Widget(const Widget&)'` - the AST makes the copy constructor call explicit even though the source just says `auto copy = w`.

### Q2: Use AST to understand overload resolution

When you're unsure which overload the compiler picks, or why a call involves an implicit conversion you didn't intend, the AST dump tells you definitively. Every `ImplicitCastExpr` node in the AST corresponds to a conversion the compiler silently inserted.

```cpp
// overload_demo.cpp
#include <iostream>
#include <string>

void process(int x)         { std::cout << "int: " << x << '\n'; }
void process(double x)      { std::cout << "double: " << x << '\n'; }
void process(std::string x) { std::cout << "string: " << x << '\n'; }

int main() {
    process(42);       // which overload?
    process(3.14);     // which overload?
    process(42.0f);    // which overload? (float -> double or int?)
    process("hello");  // which overload? (const char* -> string?)
}
```

```bash
$ clang++ -std=c++20 -Xclang -ast-dump -Xclang -ast-dump-filter=main \
    -fsyntax-only overload_demo.cpp

# AST reveals:
# CallExpr 'process'
#   `-IntegerLiteral 42          -> calls process(int)  [exact match]
#
# CallExpr 'process'
#   `-FloatingLiteral 3.14       -> calls process(double)  [exact match]
#
# CallExpr 'process'
#   `-ImplicitCastExpr <FloatingCast>   <-- float promoted to double
#     `-FloatingLiteral 42.0f    -> calls process(double)  [promotion]
#
# CallExpr 'process'
#   `-ImplicitCastExpr <UserDefinedConversion>  <-- implicit conversion
#     `-CXXConstructExpr 'std::string(const char*)'  <-- string ctor called
#       `-StringLiteral "hello"  -> calls process(string)  [user conversion]

# The AST shows EVERY implicit conversion and which overload was chosen
```

The `42.0f` case is particularly instructive. The AST shows a `FloatingCast` node - the `float` was promoted to `double`. This is why `process(double)` is called rather than `process(int)`, even though `42` and `42.0f` might look similar to a casual reader.

### Q3: AST matchers for clang-tidy custom checks

The AST is not just for inspection - it's the foundation of clang-tidy's analysis. clang-tidy custom checks work by writing a pattern (called a "matcher") that describes an AST subtree you want to detect. When the matcher fires, you can issue a diagnostic.

```cpp
// Anti-pattern to detect: calling .size() == 0 instead of .empty()
// BAD:  if (vec.size() == 0)
// GOOD: if (vec.empty())
```

Before writing a check in C++, you can prototype the matcher interactively with `clang-query`. This lets you experiment with the syntax until the matcher fires on exactly the patterns you want.

```bash
# Use clang-query to test AST matchers interactively:
$ clang-query ast_demo.cpp -- -std=c++20

clang-query> match binaryOperator(
  hasOperatorName("=="),
  hasOperandExpression(
    cxxMemberCallExpr(callee(cxxMethodDecl(hasName("size"))))
  ),
  hasOperandExpression(
    integerLiteral(equals(0))
  )
)
# Match found! Points to: vec.size() == 0
```

Once the matcher is working in `clang-query`, you translate it into a clang-tidy check. The C++ code below is what a minimal clang-tidy plugin looks like.

```cpp
// Conceptual clang-tidy check (C++ code for the check itself):
// In a clang-tidy plugin:
void SizeComparedToZero::registerMatchers(ast_matchers::MatchFinder* Finder) {
    Finder->addMatcher(
        binaryOperator(
            hasOperatorName("=="),
            hasLHS(cxxMemberCallExpr(
                callee(cxxMethodDecl(hasName("size"))))
                .bind("size_call")),
            hasRHS(integerLiteral(equals(0)))
        ).bind("comparison"),
        this);
}

void SizeComparedToZero::check(const MatchFinder::MatchResult& Result) {
    const auto* Match = Result.Nodes.getNodeAs<BinaryOperator>("comparison");
    diag(Match->getBeginLoc(),
         "use .empty() instead of .size() == 0");
}
```

The `.bind("name")` calls attach names to matched nodes so you can retrieve them in the `check()` callback. This is how clang-tidy knows the exact source location to report in the diagnostic.

---

## Notes

- Use `clang-query` to interactively test AST matchers before writing clang-tidy checks.
- `-ast-dump-filter=functionName` limits output to specific declarations.
- C++ Insights (cppinsights.io) also shows implicit conversions in a readable format.
- The AST shows implicit copy/move constructors, conversions, and template instantiations.
- Use `-ast-dump=json` for machine-parseable output.
