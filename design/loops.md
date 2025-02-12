---
author: [amisi]
---

# Loops

In approaching the problem of loops, our focus must first and foremost be in
identifying a useful subspace within C that is relatively easy to verify.
Our commitment to ensuring that all well-defined C is expressible within Xr0
must not distract us.

Our Lex parser provides the right starting point.
Greping [parse.x] and sorting, I see the following loops:

```C
for (i = 0; i < arr->n; i++) { /* ... */ }
for (i = 0; i < arr->n; i++) { /* ... */ }
for (i = 0; i < l->patterns->n; i++) { /* ... */ }
for (i = 0; i < l->tokens->n; i++) { /* ... */ }
for (i = 0; i < n; i++) { /* ... */ }
for (i = 0; i < n; i++) { /* ... */ }

for (; *pos != '\0' && strncmp(pos, "%%", 2) != 0 ; n++) { /* ... */ }
for (; *pos != '}'; pos++) {}
for (; *s != '\0'; 0) { /* ... */ }
for (; *s != '\n'; 0) { /* ... */ }
for (; *s == ' ' || *s == '\t'; s++) {}
for (; isalpha(*s) || isdigit(*s) || *s == '_' ; 0) { /* ... */ }
for (; isspace(*s); s++) {}
for (; strncmp(pos, "%%", 2) != 0; n++) { /* ... */ }
for (; strncmp(pos, "%}", 2) != 0; pos++) {}
for (pos++; *pos != '"'; pos++) {}
```

  [parse.x]: https://github.com/xr0-org/xr0/blob/master/tests/0v/99-program/100-lex/parse.x

There's far too much diversity here.
So our fast task will be identifying a uniform model for expressing loops.


## While(1) form

The essential character of a loop is repetition. 

```C
for (i = 0; i < n; i++) {
        /* do stuff */
}
```
