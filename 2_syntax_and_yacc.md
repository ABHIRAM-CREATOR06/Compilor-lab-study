# Part 2: Syntax Analyzer (C) and YACC

Run C: `gcc p.c -o p && ./p`

---

## A. Syntax Analyzer Programs (C)

### A1, A2, A3 and Misc 2. Recursive-descent parser

A left-recursive rule like `E -> E + T` loops forever in code, so remove the left recursion first:

```
E  -> T E'        E' -> + T E' | - T E' | e
T  -> F T'        T' -> * F T' | / F T' | e
F  -> ( E ) | id
```

One function per nonterminal. The `e` case is simply "do nothing".

```c
#include <stdio.h>
#include <ctype.h>
char s[50]; int p = 0, err = 0;
void E(), T(), F(), Ep(), Tp();
void E(){ T(); Ep(); }
void Ep(){ if(s[p] == '+' || s[p] == '-'){ p++; T(); Ep(); } }
void T(){ F(); Tp(); }
void Tp(){ if(s[p] == '*' || s[p] == '/'){ p++; F(); Tp(); } }
void F(){
    if(isalpha(s[p])){ while(isalnum(s[p])) p++; }
    else if(s[p] == '('){ p++; E(); if(s[p] == ')') p++; else err = 1; }
    else err = 1;
}
int main(){
    scanf("%s", s);                 /* a+b*(c-d) */
    E();
    if(!err && s[p] == '\0') puts("Valid"); else puts("Invalid");
    return 0;
}
```

For A3 (only identifiers, `+`, `*`, parentheses), delete the `'-'` in `Ep` and the `'/'` in `Tp`.

### A4 and Misc 7. Predictive parser (table driven)

Grammar: `E->TA, A->+TA|e, T->FB, B->*FB|e, F->(E)|i`
Here `i` is id, an empty string means epsilon. Input: `i+i*i`

```c
#include <stdio.h>
#include <string.h>
#include <ctype.h>
char *table(char n, char t){
    switch(n){
    case 'E': if(t == 'i' || t == '(') return "TA"; break;
    case 'A': if(t == '+') return "+TA";
              if(t == ')' || t == '$') return ""; break;
    case 'T': if(t == 'i' || t == '(') return "FB"; break;
    case 'B': if(t == '*') return "*FB";
              if(t == '+' || t == ')' || t == '$') return ""; break;
    case 'F': if(t == 'i') return "i";
              if(t == '(') return "(E)"; break;
    }
    return NULL;
}
int main(){
    char in[50], st[50], *r; int top = 1, p = 0, ok = 1, i;
    scanf("%s", in); strcat(in, "$");
    st[0] = '$'; st[1] = 'E';
    while(top >= 0 && ok){
        char X = st[top], a = in[p];
        if(X == a){ top--; p++; }                       /* match */
        else if(isupper(X)){
            r = table(X, a);
            if(!r) ok = 0;
            else { top--; for(i = strlen(r) - 1; i >= 0; i--) st[++top] = r[i]; }
        }
        else ok = 0;
    }
    puts(ok ? "Accepted" : "Rejected");
    return 0;
}
```

### A5, A6 and Misc 4. FIRST and FOLLOW

Input format: number of productions, then each as `LHS=RHS` with single uppercase letters for nonterminals and `#` for epsilon. Then the nonterminal to query.

```
8
E=TA
A=+TA
A=#
T=FB
B=*FB
B=#
F=(E)
F=i
E
```

Start symbol is the LHS of the first production. This handles the usual textbook grammars (it only looks at the first symbol of each right side, plus one nullable check in FOLLOW).

```c
#include <stdio.h>
#include <ctype.h>
#include <string.h>
char p[20][20], res[30];
int n, k, vf[128], vo[128];
void add(char c){
    if(c == '#') return;
    for(int i = 0; i < k; i++) if(res[i] == c) return;
    res[k++] = c;
}
int nullable(char c){
    for(int i = 0; i < n; i++) if(p[i][0] == c && p[i][2] == '#') return 1;
    return 0;
}
void first(char c){
    if(!isupper(c)){ add(c); return; }
    if(vf[(int)c]) return;
    vf[(int)c] = 1;
    for(int i = 0; i < n; i++) if(p[i][0] == c) first(p[i][2]);
}
void follow(char c){
    if(vo[(int)c]) return;
    vo[(int)c] = 1;
    if(c == p[0][0]) add('$');
    for(int i = 0; i < n; i++)
        for(int j = 2; p[i][j]; j++)
            if(p[i][j] == c){
                char x = p[i][j+1];
                if(x){ memset(vf, 0, sizeof vf); first(x); }
                if(!x || nullable(x)) follow(p[i][0]);
            }
}
int main(){
    char c; int i;
    scanf("%d", &n);
    for(i = 0; i < n; i++) scanf("%s", p[i]);
    scanf(" %c", &c);
    k = 0; first(c);
    printf("FIRST(%c)  = { ", c);
    for(i = 0; i < k; i++) printf("%c ", res[i]);
    if(nullable(c)) printf("# ");
    printf("}\n");
    k = 0; follow(c);
    printf("FOLLOW(%c) = { ", c);
    for(i = 0; i < k; i++) printf("%c ", res[i]);
    printf("}\n");
    return 0;
}
```

### A7 and Misc 10. Shift-reduce parser

Grammar: `E -> E+E | E*E | (E) | i`. Input: `i+i*i`

```c
#include <stdio.h>
char in[50], st[50]; int top = 0;
void reduce(){
    for(;;){
        if(top >= 1 && st[top-1] == 'i'){
            st[top-1] = 'E'; puts("   reduce E->i");
        }
        else if(top >= 3 && st[top-3] == 'E' && (st[top-2] == '+' || st[top-2] == '*') && st[top-1] == 'E'){
            st[top-3] = 'E'; top -= 2; puts("   reduce E->E op E");
        }
        else if(top >= 3 && st[top-3] == '(' && st[top-2] == 'E' && st[top-1] == ')'){
            st[top-3] = 'E'; top -= 2; puts("   reduce E->(E)");
        }
        else break;
    }
}
int main(){
    int p;
    scanf("%s", in);
    for(p = 0; in[p]; p++){
        st[top++] = in[p]; st[top] = 0;
        printf("shift %c   stack: %s\n", in[p], st);
        reduce();
        st[top] = 0;
        printf("           stack: %s\n", st);
    }
    puts((top == 1 && st[0] == 'E') ? "Accepted" : "Rejected");
    return 0;
}
```

### A8. Balanced parentheses

```c
#include <stdio.h>
int main(){
    char s[100]; int i, c = 0;
    scanf("%s", s);
    for(i = 0; s[i]; i++){
        if(s[i] == '(') c++;
        else if(s[i] == ')'){ c--; if(c < 0) break; }
    }
    puts(c == 0 ? "Balanced" : "Not balanced");
    return 0;
}
```

### A9 and Misc 9. Eliminate immediate left recursion

Input: `E=E+T|T`. Output: `E=TE'` and `E'=+TE'|#`

```c
#include <stdio.h>
#include <string.h>
int main(){
    char s[50], al[20], be[20]; int i = 3, j = 0;
    scanf("%s", s);
    if(s[2] != s[0]){ puts("No left recursion"); return 0; }
    while(s[i] && s[i] != '|') al[j++] = s[i++];
    al[j] = 0;
    strcpy(be, s + i + 1);
    printf("%c=%s%c'\n%c'=%s%c'|#\n", s[0], be, s[0], s[0], al, s[0]);
    return 0;
}
```

### A10 and Misc 8. Left factoring

Input: `A=abc|abd`. Output: `A=abA'` and `A'=c|d`

```c
#include <stdio.h>
#include <string.h>
int main(){
    char s[50], a[20], b[20]; int i = 2, j = 0, k = 0;
    scanf("%s", s);
    while(s[i] && s[i] != '|') a[j++] = s[i++];
    a[j] = 0;
    strcpy(b, s + i + 1);
    while(a[k] && a[k] == b[k]) k++;                /* common prefix length */
    if(k == 0){ puts("No common prefix"); return 0; }
    printf("%c=", s[0]);
    for(i = 0; i < k; i++) putchar(a[i]);
    printf("%c'\n%c'=%s|%s\n", s[0], s[0], a[k] ? a + k : "#", b[k] ? b + k : "#");
    return 0;
}
```

### A11 and Misc 3. String belongs to a grammar

Grammar `S -> aSb | ab` (that is a^n b^n, n >= 1). Count the a's, then the b's.

```c
#include <stdio.h>
#include <string.h>
int main(){
    char s[50]; int i = 0, n;
    scanf("%s", s);
    while(s[i] == 'a') i++;
    n = i;
    while(s[i] == 'b') i++;
    if(n > 0 && i == strlen(s) && i == 2 * n) puts("Belongs to grammar");
    else puts("Does not belong");
    return 0;
}
```

---

## B. YACC Programs

Every YACC program is two files, a `.l` that returns tokens and a `.y` with the grammar.

Run (flex and bison):
```
flex p.l
bison -y -d p.y
gcc lex.yy.c y.tab.c -o p
./p
```
With classic tools: `lex p.l`, `yacc -d p.y`, same gcc line. Type one line and press Enter.

### B1, B2, B5, B8. Calculator, expression validity, precedence and parentheses

The same pair of files covers all four. `%left` lines give precedence (lower line = lower precedence).

`calc.l`
```lex
%{
#include <stdlib.h>
#include "y.tab.h"
%}
%%
[0-9]+    { yylval = atoi(yytext); return NUM; }
[ \t]     ;
.|\n      { return yytext[0]; }
%%
int yywrap(){ return 1; }
```

`calc.y`
```yacc
%{
#include <stdio.h>
int yylex(void);
void yyerror(const char *s){ printf("Invalid expression\n"); }
%}
%token NUM
%left '+' '-'
%left '*' '/'
%%
S : E '\n'      { printf("Valid, result = %d\n", $1); }
  ;
E : E '+' E     { $$ = $1 + $3; }
  | E '-' E     { $$ = $1 - $3; }
  | E '*' E     { $$ = $1 * $3; }
  | E '/' E     { $$ = $1 / $3; }
  | '(' E ')'   { $$ = $2; }
  | NUM         { $$ = $1; }
  ;
%%
int main(){ yyparse(); return 0; }
```

For B2 (only recognise, no value) use `printf("Valid expression\n")` in the S action. To allow identifiers, add `[a-z]+ { return ID; }` to the lexer, `%token ID` and `| ID` in `E`.

### B3. Palindrome

A pure grammar cannot find the middle of a palindrome without conflicts, so the lexer passes the word and the action checks it in C.

`pal.l`
```lex
%{
#include <string.h>
#include "y.tab.h"
char buf[100];
%}
%%
[a-zA-Z0-9]+   { strcpy(buf, yytext); return WORD; }
\n             { return '\n'; }
.              ;
%%
int yywrap(){ return 1; }
```

`pal.y`
```yacc
%{
#include <stdio.h>
#include <string.h>
int yylex(void);
void yyerror(const char *s){ printf("Invalid input\n"); }
extern char buf[100];
%}
%token WORD
%%
S : WORD '\n' {
        int i, n = strlen(buf), ok = 1;
        for(i = 0; i < n / 2; i++) if(buf[i] != buf[n-1-i]) ok = 0;
        printf(ok ? "Palindrome\n" : "Not a palindrome\n");
    }
  ;
%%
int main(){ yyparse(); return 0; }
```

### B4. Valid variable declaration

Input: `int a,b,c;`

`decl.l`
```lex
%{
#include "y.tab.h"
%}
%%
int|float|char|double    { return TYPE; }
[a-zA-Z_][a-zA-Z0-9_]*   { return ID; }
[ \t]                    ;
\n                       { return '\n'; }
.                        { return yytext[0]; }
%%
int yywrap(){ return 1; }
```

`decl.y`
```yacc
%{
#include <stdio.h>
int yylex(void);
void yyerror(const char *s){ printf("Invalid declaration\n"); }
%}
%token TYPE ID
%%
S : D '\n'      { printf("Valid declaration\n"); }
  ;
D : TYPE L ';' ;
L : L ',' ID
  | ID
  ;
%%
int main(){ yyparse(); return 0; }
```

### B6. Valid if-else statement

Input: `if(a<b)x=1;else x=2;`

`ifelse.l`
```lex
%{
#include "y.tab.h"
%}
%%
if                       { return IF; }
else                     { return ELSE; }
[a-zA-Z_][a-zA-Z0-9_]*   { return ID; }
[0-9]+                   { return NUM; }
[ \t]                    ;
\n                       { return '\n'; }
.                        { return yytext[0]; }
%%
int yywrap(){ return 1; }
```

`ifelse.y`
```yacc
%{
#include <stdio.h>
int yylex(void);
void yyerror(const char *s){ printf("Invalid if-else\n"); }
%}
%token IF ELSE ID NUM
%%
S : ST '\n'   { printf("Valid if-else\n"); }
  ;
ST : IF '(' C ')' A ';' ELSE A ';' ;
C  : E R E ;
R  : '<' | '>' | '=' '=' ;
E  : ID | NUM ;
A  : ID '=' E ;
%%
int main(){ yyparse(); return 0; }
```

### B7. Input satisfies a specified grammar

Grammar `S -> aSb | ab`. Input: `aabb`

`anbn.l`
```lex
%{
#include "y.tab.h"
%}
%%
.|\n    { return yytext[0]; }
%%
int yywrap(){ return 1; }
```

`anbn.y`
```yacc
%{
#include <stdio.h>
int yylex(void);
void yyerror(const char *s){ printf("Does not satisfy the grammar\n"); }
%}
%%
T : S '\n'      { printf("Satisfies the grammar\n"); }
  ;
S : 'a' S 'b'
  | 'a' 'b'
  ;
%%
int main(){ yyparse(); return 0; }
```
