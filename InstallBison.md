# Introduction #
由于系统利用Bison来生成访问缺省的系统配置文件"/etc/otpd.conf"的程序，当需要修改config.y文件，并重新生成y.tab.c y.tab.h y.tab.o文件的时候，需要在开发环境上安装并利用Bison和flex。

# Details #
系统相关文件一览
  1. config.y  //给Bison的输入文件(Bison grammar file)
  1. y.tab.c   //Bison的输出文件,包含yyparse()函数
  1. y.tab.h    //Bison的输出文件,为了在scanner.c里的利用而单独生成的头文件
  1. config.l   //给flex的输入文件(a description of a scanner)
  1. scanner.c    //flex的输出文件(lexical scanner),包含yylex()函数
  1. scanner.o    //编译后文件
  1. y.tab.o      //编译后文件

下载并安装Bison
```
# wget ftp://ftp.gnu.org/gnu/bison/bison-2.4.3.tar.gz
# gzip -dc bison-2.4.3.tar.gz | tar xf -
# cd bison-2.4.3
# ./configure
# make
# make install
```

bison的详细内容请参照 http://www.gnu.org/software/bison/。
为了测试Bison的运行，可以参照 http://www.gnu.org/software/bison/manual/bison.html#RPN-Calc
里面的例子 rpcalc.y(一个逆波兰式表达式计算器)来生成C语言代码，编译并执行。
```
     /* Reverse polish notation calculator.  */
     
     %{
       #define YYSTYPE double
       #include <math.h>
       #include <stdio.h>
       int yylex (void);
       void yyerror (char const *);
     %}
     
     %token NUM

     %% /* Grammar rules and actions follow.  */

     input:    /* empty */
             | input line
     ;
     
     line:     '\n'
             | exp '\n'      { printf ("\t%.10g\n", $1); }
     ;
     
     exp:      NUM           { $$ = $1;           }
             | exp exp '+'   { $$ = $1 + $2;      }
             | exp exp '-'   { $$ = $1 - $2;      }
             | exp exp '*'   { $$ = $1 * $2;      }
             | exp exp '/'   { $$ = $1 / $2;      }
              /* Exponentiation */
             | exp exp '^'   { $$ = pow ($1, $2); }
              /* Unary minus    */
             | exp 'n'       { $$ = -$1;          }
     ;
     %%

     /* The lexical analyzer returns a double floating point
        number on the stack and the token NUM, or the numeric code
        of the character read if not a number.  It skips all blanks
        and tabs, and returns 0 for end-of-input.  */
     
     #include <ctype.h>
     
     int
     yylex (void)
     {
       int c;
     
       /* Skip white space.  */
       while ((c = getchar ()) == ' ' || c == '\t')
         ;
       /* Process numbers.  */
       if (c == '.' || isdigit (c))
         {
           ungetc (c, stdin);
           scanf ("%lf", &yylval);
           return NUM;
         }
       /* Return end-of-input.  */
       if (c == EOF)
         return 0;
       /* Return a single char.  */
       return c;
     }

     int
     main (void)
     {
       return yyparse ();
     }

     #include <stdio.h>
     
     /* Called by yyparse on error.  */
     void
     yyerror (char const *s)
     {
       fprintf (stderr, "%s\n", s);
     }
```
运行以下命令把rpcalc.y转化成解析文件
```
     # bison rpcalc.y
```
编译
```
     ## 显示当前文件
     # ls
     rpcalc.tab.c  rpcalc.y
     
     ## 编译Bison parser.
     ##‘-lm’ tells compiler to search math library for pow.
     # cc -lm -o rpcalc rpcalc.tab.c
     
     ## 编译后在此显示当前文件
     # ls
     rpcalc  rpcalc.tab.c  rpcalc.y
```
运行
```
     #./rpcalc
      4 0 +                    //输入
	4                      //运算结果
      3 7 + 3 4 5 *+           //输入
	-13                    //运算结果
      3 7 + 3 4 5 * + - n      //输入
	13                     //运算结果
      5 6 / 4 n +              //输入
	-3.166666667           //运算结果
```