# Part 1: Lexical Analyzer (C) and LEX

Run C: `gcc p.c -o p && ./p`
Run LEX: `flex p.l && gcc lex.yy.c -o p && ./p` (end keyboard input with Ctrl+D)

---

## A. Lexical Analyzer Programs (C)

### A1 and A8. Lexical analyzer (keywords, identifiers, operators, constants, special symbols), input from a file

Create `input.c` with any C code. To read the keyboard instead, use `FILE *f = stdin;`.

```c
#include <stdio.h>
#include <string.h>
#include <ctype.h>
char *kw[] = {"int","float","char","if","else","while","for","return","void","double"};
int iskw(char *s){
    for(int i = 0; i < 10; i++) if(!strcmp(s, kw[i])) return 1;
    return 0;
}
int main(){
    FILE *f = fopen("input.c", "r");
    int c, i; char b[50];
    while((c = fgetc(f)) != EOF){
        if(isspace(c)) continue;
        if(isalpha(c) || c == '_'){
            i = 0; b[i++] = c;
            while(isalnum(c = fgetc(f)) || c == '_') b[i++] = c;
            b[i] = 0; ungetc(c, f);
            printf("%s : %s\n", b, iskw(b) ? "Keyword" : "Identifier");
        }
        else if(isdigit(c)){
            i = 0; b[i++] = c;
            while(isdigit(c = fgetc(f)) || c == '.') b[i++] = c;
            b[i] = 0; ungetc(c, f);
            printf("%s : Constant\n", b);
        }
        else if(strchr("+-*/%=<>!&|", c)) printf("%c : Operator\n", c);
        else printf("%c : Special symbol\n", c);
    }
    return 0;
}
```

### A2 and A5. Count keywords, identifiers, operators (and total tokens) in a statement

Input: `int a = b + 5;`

```c
#include <stdio.h>
#include <string.h>
#include <ctype.h>
int main(){
    char s[100], b[30];
    char *k[] = {"int","float","char","if","else","while","for","return"};
    int i = 0, j, kw = 0, id = 0, op = 0, tok = 0;
    fgets(s, 100, stdin);
    while(s[i]){
        if(isalpha(s[i]) || s[i] == '_'){
            j = 0;
            while(isalnum(s[i]) || s[i] == '_') b[j++] = s[i++];
            b[j] = 0; tok++;
            for(j = 0; j < 8; j++) if(!strcmp(b, k[j])) break;
            if(j < 8) kw++; else id++;
        }
        else if(isdigit(s[i])){
            while(isdigit(s[i]) || s[i] == '.') i++;
            tok++;
        }
        else if(strchr("+-*/%=<>!&|", s[i])){ op++; tok++; i++; }
        else { if(!isspace(s[i])) tok++; i++; }
    }
    printf("Keywords=%d Identifiers=%d Operators=%d Tokens=%d\n", kw, id, op, tok);
    return 0;
}
```

### A3. Valid C identifier

```c
#include <stdio.h>
#include <ctype.h>
int main(){
    char s[50]; int i, ok = 1;
    scanf("%s", s);
    if(!(isalpha(s[0]) || s[0] == '_')) ok = 0;
    for(i = 1; s[i]; i++)
        if(!(isalnum(s[i]) || s[i] == '_')) ok = 0;
    puts(ok ? "Valid identifier" : "Invalid identifier");
    return 0;
}
```

### A4. Recognize arithmetic, relational, logical operators

```c
#include <stdio.h>
#include <string.h>
#define EQ(x) !strcmp(s, x)
int main(){
    char s[5];
    scanf("%s", s);
    if(EQ("+") || EQ("-") || EQ("*") || EQ("/") || EQ("%")) puts("Arithmetic");
    else if(EQ("<") || EQ(">") || EQ("<=") || EQ(">=") || EQ("==") || EQ("!=")) puts("Relational");
    else if(EQ("&&") || EQ("||") || EQ("!")) puts("Logical");
    else puts("Not an operator");
    return 0;
}
```

### A6. Integer and floating-point constants

```c
#include <stdio.h>
#include <ctype.h>
int main(){
    char s[30]; int i, dot = 0, ok = 1;
    scanf("%s", s);
    for(i = 0; s[i]; i++){
        if(s[i] == '.') dot++;
        else if(!isdigit(s[i])) ok = 0;
    }
    if(!ok || dot > 1) puts("Not a constant");
    else if(dot == 1) puts("Floating-point constant");
    else puts("Integer constant");
    return 0;
}
```

### A7. Remove comments from a C program

Create `in.c` containing comments.

```c
#include <stdio.h>
int main(){
    FILE *f = fopen("in.c", "r");
    int c, d;
    while((c = fgetc(f)) != EOF){
        if(c == '/'){
            d = fgetc(f);
            if(d == '/'){                         /* single line */
                while((c = fgetc(f)) != '\n' && c != EOF);
                putchar('\n');
            }
            else if(d == '*'){                    /* multi line */
                while((c = fgetc(f)) != EOF){
                    if(c == '*'){
                        d = fgetc(f);
                        if(d == '/') break;
                        ungetc(d, f);
                    }
                }
            }
            else { putchar(c); ungetc(d, f); }
        }
        else putchar(c);
    }
    return 0;
}
```

---

## B. LEX Programs

Every LEX file has the same shape:

```
%{  C headers and variables  %}
%%
  pattern   { action }
%%
int main(){ yylex(); ... }
int yywrap(){ return 1; }
```

### B1. Count characters, words, spaces, lines

```lex
%{
#include <stdio.h>
int ch = 0, w = 0, sp = 0, ln = 0;
%}
%%
[ \t]       { sp++; ch++; }
\n          { ln++; ch++; }
[^ \t\n]+   { w++; ch += yyleng; }
%%
int main(){
    yylex();
    printf("Chars=%d Words=%d Spaces=%d Lines=%d\n", ch, w, sp, ln);
    return 0;
}
int yywrap(){ return 1; }
```

### B2. Keywords, identifiers, constants, operators

```lex
%{
#include <stdio.h>
%}
%%
int|float|char|if|else|while|for|return|void   { printf("%s : keyword\n", yytext); }
[a-zA-Z_][a-zA-Z0-9_]*                         { printf("%s : identifier\n", yytext); }
[0-9]+(\.[0-9]+)?                              { printf("%s : constant\n", yytext); }
[+\-*/=<>!&|%]+                                { printf("%s : operator\n", yytext); }
.|\n                                           ;
%%
int main(){ yylex(); return 0; }
int yywrap(){ return 1; }
```

### B3 and B6. Valid identifier (type one string per line)

```lex
%{
#include <stdio.h>
%}
%%
^[a-zA-Z_][a-zA-Z0-9_]*$   { printf("Valid identifier\n"); }
.*                         { printf("Invalid identifier\n"); }
\n                         ;
%%
int main(){ yylex(); return 0; }
int yywrap(){ return 1; }
```

### B4. Count vowels and consonants

Type one line. Enter ends the input.

```lex
%{
#include <stdio.h>
int v = 0, c = 0;
%}
%%
[aeiouAEIOU]   v++;
[a-zA-Z]       c++;
\n             { return 0; }
.              ;
%%
int main(){
    yylex();
    printf("Vowels=%d Consonants=%d\n", v, c);
    return 0;
}
int yywrap(){ return 1; }
```

### B5. Integers and floating-point numbers

```lex
%{
#include <stdio.h>
%}
%%
[0-9]+            { printf("%s : integer\n", yytext); }
[0-9]*\.[0-9]+    { printf("%s : float\n", yytext); }
.|\n              ;
%%
int main(){ yylex(); return 0; }
int yywrap(){ return 1; }
```

### B7. Relational operators

```lex
%{
#include <stdio.h>
%}
%%
"<="|">="|"=="|"!="|"<"|">"   { printf("%s : relational operator\n", yytext); }
.|\n                          ;
%%
int main(){ yylex(); return 0; }
int yywrap(){ return 1; }
```

### B8. Remove single-line and multiline comments

Run it as `./p < in.c`. Anything not matched is echoed by default, so only the comments disappear.

```lex
%%
"//".*                            ;
"/*"([^*]|\*+[^*/])*\*+"/"        ;
%%
int main(){ yylex(); return 0; }
int yywrap(){ return 1; }
```

### B9. Count arithmetic operators

```lex
%{
#include <stdio.h>
int n = 0;
%}
%%
[+\-*/%]   n++;
.|\n       ;
%%
int main(){
    yylex();
    printf("Arithmetic operators = %d\n", n);
    return 0;
}
int yywrap(){ return 1; }
```

### B10. Input contains only digits

```lex
%{
#include <stdio.h>
%}
%%
^[0-9]+$   { printf("Only digits\n"); }
.*         { printf("Not only digits\n"); }
\n         ;
%%
int main(){ yylex(); return 0; }
int yywrap(){ return 1; }
```
