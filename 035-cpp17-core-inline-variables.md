## inline変数

C++17では変数にinlineキーワードを指定できるようになった。

~~~cpp
inline int variable ;
~~~

このような変数をinline変数と呼ぶ。その意味はinline関数と同じだ。

### inlineの歴史的な意味

今は昔、本書執筆から30年以上は昔に、inlineキーワードがC++に追加された。

inlineの現在の意味は誤解されている。

inline関数の意味は、「関数を強制的にインライン展開させるための機能」**ではない**。

大事なことなのでもう一度書くが、inline関数の意味は、「関数を強制的にインライン展開させるための機能」**ではない**。

確かに、かつてinline関数の意味は、関数を強制的にインライン展開させるための機能だった。

関数のインライン展開とは、例えば以下のようなコードがあったとき、

~~~cpp
int min( int a, int b )
{ return a < b ? a : b ; }

int main()
{
    int a, b ;
    std::cin >> a >> b ;

    // aとbのうち小さい方を選ぶ
    int value = min( a, b ) ;
}
~~~

この関数minは十分に小さく、関数呼び出しのコストは無視できないオーバーヘッドになるため、以下のような最適化が考えられる。

~~~cpp
int main()
{
    int a, b ;
    std::cin >> a >> b ;

    int value = a < b ? a : b ;
}
~~~

このように関数の中身を展開することを、関数のインライン展開という。

人間が関数のインライン展開を手で行うのは面倒だ。それにコードが読みにくい。"min(a,b)"と"a\<b?a:b"のどちらが読みやすいだろうか。

幸い、C++コンパイラーはインライン展開を自動的に行えるので人間が苦労する必要はない。

インライン展開は万能の最適化ではない。インライン展開をすると逆に遅くなる場合もある。

例えば、ある関数をコンパイルした結果のコードサイズが1KBあったとして、その関数を呼んでいる箇所がプログラム中に1000件ある場合、プログラム全体のサイズは1MB増える。コードサイズが増えるということは、CPUのキャッシュを圧迫する。

例えば、ある関数の実行時間が関数呼び出しの実行時間に比べて桁違いに長い時、関数呼び出しのコストを削減するのは意味がない。

したがって関数のインライン展開という最適化を適用すべきかどうかを決定するには、関数のコードサイズが十分に小さい時、関数の実行時間が十分に短い時、タイトなループの中など、様々な条件を考慮しなければならない。

昔のコンパイラー技術が未熟だった時代のC++コンパイラーは関数をインライン展開するべきかどうかの判断ができなかった。そのためinlineキーワードが追加された。インライン展開してほしい関数をinline関数にすることで、コンパイラーはその関数がインライン展開するべき関数だと認識する。

### 現代のinlineの意味

現代では、コンパイラー技術の発展によりC++コンパイラーは十分に賢くなったので、関数をインライン展開させる目的でinlineキーワードを使う必要はない。実際、現代のC++コンパイラーはinlineキーワードを無視する。関数をインライン展開すべきかどうかはコンパイラーが判断できる。

inlineキーワードにはインライン展開以外に、もうひとつの意味がある。ODR(One Definition Rule、定義はひとつの原則)の回避だ。

C++では、定義はプログラム中にひとつしか書くことができない。

~~~c++
void f() ; // OK、宣言
void f() ; // OK、再宣言

void f() { } // OK、定義

void f() { } // エラー、再定義
~~~

通常は、関数を使う場合には宣言だけを書いて使う。定義はどこかひとつの翻訳単位に書いておけばよい。

~~~c++
// f.h

void f() ;

// f.cpp

void f() { }

// main.cpp

#include "f.h"

int main()
{
    f() ;
}
~~~

しかし、関数のインライン展開をするには、コンパイラーの実装上の都合で、関数の定義が同じ翻訳単位になければならない。


~~~c++
inline void f() ;

int main()
{
    // エラー、定義がない
    f() ; 
}
~~~

しかし、翻訳単位ごとに定義すると、定義が重複してODRに違反する。

C++ではこの問題を解決するために、inline関数は定義が同一であれば、複数の翻訳単位で定義されてもよいことにしている。つまりODRに違反しない。

~~~c++
// a.cpp

inline void f() { }

void a()
{
    f() ;
}

// b.cpp

// OK、inline関数
inline void f() { }

void b()
{
    f() ;
}
~~~

これは例のために同一のinline関数を直接記述しているが、inline関数は定義を同一性を保証させるため、通常はヘッダーファイルに書いて#includeして使う。

### inline変数の意味

inline変数は、ODRに違反せず変数の定義の重複を認める。同じ名前のinline変数は同じ変数を指す。

~~~c++
// a.cpp

inline int data ;

void a() { ++data ; }

// b.cpp

inline int data ;

void b() { ++data ; }

// main.cpp

inline int data ;

int main()
{
    a() ;
    b() ;

    data ; // 2
}
~~~

この例で関数a, bの中の変数dataは同じ変数を指している。変数dataはstaticストレージ上に構築された変数なのでプログラムの開始時にゼロで初期化される。2回インクリメントされるので値は2となる。

これにより、クラスの非staticデータメンバーの定義を書かなくてすむようになる。

C++17以前のC++では、以下のように書かなければならなかったが、

~~~cpp
// S.h

struct S
{
    static int data ;
} ;

// S.cpp

int S::data ;
~~~

C++17では、以下のように書けばよい。


~~~cpp
// S.h

struct S
{
    inline static int data ;
} ;
~~~

S.cppに変数S::dataの定義を書く必要はない。

機能テストマクロは__cpp_inline_variables, 値は201606。