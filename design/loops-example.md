# Algebraic analysis of a simpler loop

```C
void
unit(int limit) ~ [ setup: limit = [0?]; ]
{
	int i;

	for (i = 0; i < limit; i++) ~ [ i = [0?limit+1]; ]
		;

	~ [ i == limit; ]
}
```

```
text:
-->	i = 0;
	while (1) ~ [
		i = $unit:1[0?limit+1];
	]{
		if (!(i<limit)) break;
		;
		i++;
	}

rconst:
	#0: int:{0?50}  "unit:0:{}"

stack:
	---------------------------- inter
	0: |1| {0:<rconst:#0>} (int limit, π)
	1: |1| {0:<>}          (int i)
	---------------------------- unit
```

```
text:
	i = 0;
-->	while (1) ~ [
		i = $unit:1[0?limit+1];
	]{
		if (!(i<limit)) break;
		;
		i++;
	}

rconst:
	#0: int:{0?50}  "unit:0:{}"

stack:
	---------------------------- inter
	0: |1| {0:<rconst:#0>}	(int limit, π)
	1: |1| {0:<int:0>}      (int i)
	---------------------------- unit
```

Setup verification establishes that `i` satisfies the invariant.

```
text:
-->	if (!(i<limit)) break;
	;
	i++;

rconst:
	#0: int:{0?50}  "unit:0:{}"
	#1: int:{0?50}  "unit:1:{}"

	#1 < #0 + 1	"i<limit+1"

stack:
	---------------------------- loop
	---------------------------- inter
	0: |1| {0:<rconst:#0>} (int limit, π)
	1: |1| {0:<rconst:#1>} (int i)
	---------------------------- unit
```

Split into conditions that satisfy `i < limit` and `!(i < limit)`.

### `i < limit`

```
text:
	if (!(i<limit)) break;
-->	;
	i++;

rconst:
	#0: int:{1?50}  "unit:0:{}"
	#1: int:{0?49}  "unit:1:{}"

	#1 < #0 	"i<limit"

stack:
	---------------------------- loop
	---------------------------- inter
	0: |1| {0:<rconst:#0>} (int limit, π)
	1: |1| {0:<rconst:#1>} (int i)
	---------------------------- unit | #1 < #0
```

```
text:
	if (!(i<limit)) break;
	;
-->	i++;

rconst:
	#0: int:{1?50}  "unit:0:{}"
	#1: int:{0?49}  "unit:1:{}"

	#1 < #0 	"i<limit"

stack:
	---------------------------- loop
	---------------------------- inter
	0: |1| {0:<rconst:#0>} (int limit, π)
	1: |1| {0:<rconst:#1>} (int i)
	---------------------------- unit | #1 < #0
```

```
text:
	<end of frame>

rconst:
	#0: int:{1?50}  "unit:0:{}"
	#1: int:{0?49}  "unit:1:{}"

	#1 < #0 	"i<limit"

stack:
	---------------------------- loop
	---------------------------- inter
	0: |1| {0:<rconst:#0>}		(int limit, π)
	1: |1| {0:<rconst:#1+1>}	(int i)
	---------------------------- unit | #1 < #0
```

Checking invariant reduces to showing that `#1 + 1 < #0 + 1`, which is
equivalent to the formula `#1 < #0`. So there needs to be the capacity to deduce
this fact.

### `!(i < limit)`

```
text:
-->	if (!(i<limit)) break;
	;
	i++;

rconst:
	#0: int:{0?50}  "unit:0:{}"
	#1: int:{0?50}  "unit:1:{}"

	#1 == #0 	"i==limit"

stack:
	---------------------------- loop
	---------------------------- inter
	0: |1| {0:<rconst:#0>} (int limit, π)
	1: |1| {0:<rconst:#1>} (int i)
	---------------------------- unit | !(#1 < #0)
```
