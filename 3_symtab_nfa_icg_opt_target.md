# Part 3: Symbol Table, NFA/DFA, Intermediate Code, Optimization, Target Code

Run: `gcc p.c -o p && ./p`

Duplicates in the question list:
- Recursive descent, string in grammar, FIRST/FOLLOW, predictive parser, left factoring, left recursion, shift-reduce are all in Part 2.
- Infix to postfix is C5 below. ICG C1, C2, C3, C4, C6, C7 are one program.

---

## A. Symbol Table and Automata

### A1. Symbol table (insert, delete, search, display)

```c
#include <stdio.h>
#include <string.h>
struct { char name[20], type[10]; } t[50];
int n = 0;
int find(char *s){
    for(int i = 0; i < n; i++) if(!strcmp(t[i].name, s)) return i;
    return -1;
}
int main(){
    int ch, i; char s[20], ty[10];
    while(1){
        printf("1.Insert 2.Delete 3.Search 4.Display 5.Exit: ");
        scanf("%d", &ch);
        if(ch == 1){
            scanf("%s %s", s, ty);
            if(find(s) >= 0) puts("Already exists");
            else { strcpy(t[n].name, s); strcpy(t[n].type, ty); n++; }
        }
        else if(ch == 2){
            scanf("%s", s); i = find(s);
            if(i < 0) puts("Not found");
            else { for(; i < n - 1; i++) t[i] = t[i+1]; n--; }
        }
        else if(ch == 3){
            scanf("%s", s); i = find(s);
            if(i < 0) puts("Not found");
            else printf("Found at %d, type %s\n", i, t[i].type);
        }
        else if(ch == 4){
            for(i = 0; i < n; i++) printf("%s\t%s\n", t[i].name, t[i].type);
        }
        else break;
    }
    return 0;
}
```

### A5. NFA from a regular expression (Thompson's construction)

Input is the regex in postfix, with `.` for concatenation, `|` for union, `*` for star. `#` means epsilon in the output.

Example: `(a|b)*abb` is written `ab|*a.b.b.`

Each symbol makes two states. `.` joins two pieces, `|` and `*` add a new start and end.

```c
#include <stdio.h>
struct frag { int s, e; } st[50];
int top = 0, n = 0;
void edge(int a, char c, int b){ printf("q%d --%c--> q%d\n", a, c, b); }
int main(){
    char re[50]; int i, s, e; struct frag a, b;
    scanf("%s", re);
    for(i = 0; re[i]; i++){
        char c = re[i];
        if(c == '.'){
            b = st[--top]; a = st[--top];
            edge(a.e, '#', b.s);
            st[top].s = a.s; st[top++].e = b.e;
        }
        else if(c == '|'){
            b = st[--top]; a = st[--top]; s = n++; e = n++;
            edge(s, '#', a.s); edge(s, '#', b.s);
            edge(a.e, '#', e); edge(b.e, '#', e);
            st[top].s = s; st[top++].e = e;
        }
        else if(c == '*'){
            a = st[--top]; s = n++; e = n++;
            edge(s, '#', a.s); edge(s, '#', e);
            edge(a.e, '#', a.s); edge(a.e, '#', e);
            st[top].s = s; st[top++].e = e;
        }
        else {
            s = n++; e = n++; edge(s, c, e);
            st[top].s = s; st[top++].e = e;
        }
    }
    printf("Start: q%d   Final: q%d\n", st[0].s, st[0].e);
    return 0;
}
```

### A6. NFA to DFA (subset construction)

NFA has no epsilon moves, symbols are `a` and `b`, start state is q0. Each DFA state is a bitmask of NFA states.

Input: number of states. For each state, the targets on `a` (count, then targets), then on `b`. Then the final state.

Example NFA for `(a|b)*abb`:
```
4
2 0 1   1 0
0       1 2
0       1 3
0       0
3
```

```c
#include <stdio.h>
int n, fin, d = 1, nfa[10][2], dfa[100];
int move(int set, int c){
    int i, r = 0;
    for(i = 0; i < n; i++) if(set & (1 << i)) r |= nfa[i][c];
    return r;
}
void show(int set){
    int i; printf("{");
    for(i = 0; i < n; i++) if(set & (1 << i)) printf("q%d ", i);
    printf("}");
}
int main(){
    int i, j, c, k, x, t;
    scanf("%d", &n);
    for(i = 0; i < n; i++)
        for(c = 0; c < 2; c++){
            scanf("%d", &k);
            while(k--){ scanf("%d", &x); nfa[i][c] |= 1 << x; }
        }
    scanf("%d", &fin);
    dfa[0] = 1;                                   /* start = {q0} */
    for(i = 0; i < d; i++)
        for(c = 0; c < 2; c++){
            t = move(dfa[i], c);
            show(dfa[i]); printf(" --%c--> ", 'a' + c); show(t); printf("\n");
            for(j = 0; j < d && dfa[j] != t; j++);
            if(j == d && t) dfa[d++] = t;         /* new DFA state */
        }
    printf("Final states: ");
    for(i = 0; i < d; i++) if(dfa[i] & (1 << fin)) show(dfa[i]);
    printf("\n");
    return 0;
}
```

---

## B. Intermediate Code Generation

### C5. Infix to postfix

Input: `(a+b)*(c-d)`. Output: `ab+cd-*`

```c
#include <stdio.h>
#include <ctype.h>
int prec(char c){ return (c == '*' || c == '/') ? 2 : (c == '+' || c == '-') ? 1 : 0; }
int main(){
    char in[50], out[50], st[50]; int top = -1, k = 0, i;
    scanf("%s", in);
    for(i = 0; in[i]; i++){
        char c = in[i];
        if(isalnum(c)) out[k++] = c;
        else if(c == '(') st[++top] = c;
        else if(c == ')'){ while(st[top] != '(') out[k++] = st[top--]; top--; }
        else {
            while(top >= 0 && prec(st[top]) >= prec(c)) out[k++] = st[top--];
            st[++top] = c;
        }
    }
    while(top >= 0) out[k++] = st[top--];
    out[k] = 0;
    printf("%s\n", out);
    return 0;
}
```

### C1, C2, C3, C4, C6, C7. Three-address code, quadruples, triples, indirect triples

Input: an assignment with single-letter operands, e.g. `x=(a+b)*(c-d)`.

Idea: infix to postfix, then walk the postfix with a stack. Every operator creates a temporary `t1, t2, ...`.

Expected output for the example:
```
t1 = a + b
t2 = c - d
t3 = t1 * t2
x = t3
```

```c
#include <stdio.h>
#include <string.h>
#include <ctype.h>
#include <stdlib.h>
int prec(char c){ return (c == '*' || c == '/') ? 2 : (c == '+' || c == '-') ? 1 : 0; }
void pr(char *a){                                  /* temp t3 prints as (3) */
    if(strlen(a) > 1) printf("(%d)", atoi(a + 1)); else printf("%s", a);
}
int main(){
    char in[50], post[50], st[50], s[20][5], a1[20][5], a2[20][5], o[20];
    int top = -1, k = 0, sp = 0, n = 0, i;
    scanf("%s", in);

    for(i = 2; in[i]; i++){                        /* infix to postfix */
        char c = in[i];
        if(isalnum(c)) post[k++] = c;
        else if(c == '(') st[++top] = c;
        else if(c == ')'){ while(st[top] != '(') post[k++] = st[top--]; top--; }
        else {
            while(top >= 0 && prec(st[top]) >= prec(c)) post[k++] = st[top--];
            st[++top] = c;
        }
    }
    while(top >= 0) post[k++] = st[top--];
    post[k] = 0;

    for(i = 0; post[i]; i++){                      /* postfix to temporaries */
        if(isalnum(post[i])){ s[sp][0] = post[i]; s[sp][1] = 0; sp++; }
        else {
            n++;
            strcpy(a2[n], s[--sp]);
            strcpy(a1[n], s[--sp]);
            o[n] = post[i];
            sprintf(s[sp++], "t%d", n);
        }
    }

    printf("\nThree address code\n");
    for(i = 1; i <= n; i++) printf("t%d = %s %c %s\n", i, a1[i], o[i], a2[i]);
    printf("%c = %s\n", in[0], s[0]);

    printf("\nQuadruples\nop\targ1\targ2\tresult\n");
    for(i = 1; i <= n; i++) printf("%c\t%s\t%s\tt%d\n", o[i], a1[i], a2[i], i);
    printf("=\t%s\t-\t%c\n", s[0], in[0]);

    printf("\nTriples\n#\top\targ1\targ2\n");
    for(i = 1; i <= n; i++){
        printf("(%d)\t%c\t", i, o[i]); pr(a1[i]); printf("\t"); pr(a2[i]); printf("\n");
    }
    printf("(%d)\t=\t%c\t", n + 1, in[0]); pr(s[0]); printf("\n");

    printf("\nIndirect triples (statement list pointing to the triples above)\n");
    for(i = 1; i <= n + 1; i++) printf("%d -> (%d)\n", 100 + i - 1, i);
    return 0;
}
```

---

## C. Code Optimization

All of these read `n` and then `n` statements of the form `x=a+b` (single-letter names, one operator).

### D1. Constant folding

Input: `4 * 5`

```c
#include <stdio.h>
int main(){
    int a, b, r = 0; char op;
    scanf("%d %c %d", &a, &op, &b);
    switch(op){
        case '+': r = a + b; break;
        case '-': r = a - b; break;
        case '*': r = a * b; break;
        case '/': r = a / b; break;
    }
    printf("Before: %d %c %d\nAfter : %d\n", a, op, b, r);
    return 0;
}
```

### D2 and D5. Common subexpression and redundant computation elimination

Input:
```
3
t=a+b
u=a+b
v=t*u
```
Output: `t=a+b`, `u=t`, `v=t*u`. Assumes the operands are not reassigned in between.

```c
#include <stdio.h>
#include <string.h>
int main(){
    char s[10][20]; int n, i, j;
    scanf("%d", &n);
    for(i = 0; i < n; i++) scanf("%s", s[i]);
    for(i = 0; i < n; i++){
        for(j = 0; j < i; j++)
            if(strcmp(s[i] + 2, s[j] + 2) == 0){   /* same right side */
                s[i][2] = s[j][0]; s[i][3] = 0;    /* x = earlier result */
                break;
            }
        printf("%s\n", s[i]);
    }
    return 0;
}
```

### D3. Constant propagation

Input:
```
3
a=5
b=a+2
c=b*a
```
Output: `a=5`, `b=5+2`, `c=b*5`. Single-digit constants.

```c
#include <stdio.h>
#include <ctype.h>
int main(){
    char s[10][20], r[20], val[128] = {0};
    int n, i, j, k;
    scanf("%d", &n);
    for(i = 0; i < n; i++){
        scanf("%s", s[i]);
        k = 0;
        for(j = 2; s[i][j]; j++){
            char c = s[i][j];
            r[k++] = val[(int)c] ? val[(int)c] : c;   /* replace known constants */
        }
        r[k] = 0;
        printf("%c=%s\n", s[i][0], r);
        val[(int)s[i][0]] = (isdigit(r[0]) && !r[1]) ? r[0] : 0;
    }
    return 0;
}
```

### D4. Dead code elimination

A statement is dead if its result is never used by a later live statement. The last statement is always kept.

Input:
```
4
a=b+c
d=a+1
e=b*c
f=e+1
```
Output: `e=b*c`, `f=e+1`

```c
#include <stdio.h>
#include <string.h>
int main(){
    char s[10][20]; int n, i, j, live[10] = {0};
    scanf("%d", &n);
    for(i = 0; i < n; i++) scanf("%s", s[i]);
    live[n-1] = 1;
    for(i = n - 2; i >= 0; i--)
        for(j = i + 1; j < n; j++)
            if(live[j] && strchr(s[j] + 2, s[i][0])) live[i] = 1;
    for(i = 0; i < n; i++) if(live[i]) printf("%s\n", s[i]);
    return 0;
}
```

### D6. Algebraic simplification

Input: `x=y+0`

```c
#include <stdio.h>
int main(){
    char s[20], x, y, op, z;
    scanf("%s", s);
    x = s[0]; y = s[2]; op = s[3]; z = s[4];
    if(((op == '+' || op == '-') && z == '0') || ((op == '*' || op == '/') && z == '1'))
        printf("%c=%c\n", x, y);                       /* y+0, y-0, y*1, y/1 */
    else if((op == '+' && y == '0') || (op == '*' && y == '1'))
        printf("%c=%c\n", x, z);                       /* 0+z, 1*z */
    else if(op == '*' && (y == '0' || z == '0'))
        printf("%c=0\n", x);                           /* y*0 */
    else if(op == '*' && z == '2')
        printf("%c=%c+%c\n", x, y, y);                 /* y*2 -> y+y */
    else
        printf("%s\n", s);
    return 0;
}
```

### D7. Basic block optimization

Applies common subexpression elimination, then dead code elimination, to one block.

Input:
```
4
t=a+b
u=a+b
v=t*u
w=v+d
```
Output: `t=a+b`, `u=t`, `v=t*u`, `w=v+d`

```c
#include <stdio.h>
#include <string.h>
int main(){
    char s[10][20]; int n, i, j, live[10] = {0};
    scanf("%d", &n);
    for(i = 0; i < n; i++) scanf("%s", s[i]);
    for(i = 0; i < n; i++)                             /* step 1: common subexpressions */
        for(j = 0; j < i; j++)
            if(strcmp(s[i] + 2, s[j] + 2) == 0){ s[i][2] = s[j][0]; s[i][3] = 0; break; }
    live[n-1] = 1;                                     /* step 2: dead code */
    for(i = n - 2; i >= 0; i--)
        for(j = i + 1; j < n; j++)
            if(live[j] && strchr(s[j] + 2, s[i][0])) live[i] = 1;
    for(i = 0; i < n; i++) if(live[i]) printf("%s\n", s[i]);
    return 0;
}
```

---

## D. Target Code Generation

### E1, E2, E3, E4, E5. Three-address code to register-based target code

Input is three-address statements, either `x=y op z` or a plain copy `x=y`. Each statement gets its own register `R0, R1, ...`.

Example for `a = b + c * d`. Its three-address code is `t=c*d`, `u=b+t`, `a=u`:
```
3
t=c*d
u=b+t
a=u
```
Output:
```
MOV c,R0
MUL d,R0
MOV R0,t
MOV b,R1
ADD t,R1
MOV R1,u
MOV u,R2
MOV R2,a
```

```c
#include <stdio.h>
#include <string.h>
int main(){
    char s[10][20], ops[] = "+-*/", *ins[] = {"ADD", "SUB", "MUL", "DIV"};
    int n, i;
    scanf("%d", &n);
    for(i = 0; i < n; i++){
        scanf("%s", s[i]);
        if(strlen(s[i]) == 3)                          /* x=y */
            printf("MOV %c,R%d\nMOV R%d,%c\n", s[i][2], i, i, s[i][0]);
        else                                           /* x=y op z */
            printf("MOV %c,R%d\n%s %c,R%d\nMOV R%d,%c\n",
                   s[i][2], i, ins[strchr(ops, s[i][3]) - ops], s[i][4], i, i, s[i][0]);
    }
    return 0;
}
```
