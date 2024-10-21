---
published: Dec 22, 2023
author: [team]
---

# #5: Vincemus

Of the 12 functions that directly invoke an axiomatic memory function, we
managed to verify 9. The exceptions are due to unforeseen complexity in handling
conditional states (due to selection statements and also to allocating
statements within loops that depend on the evaluation of the loop conditional).

Since our goal was to verify all but one, we unambiguously failed to meet this
goal. Moreover, we did not iterate further on the simplification of the map and
array data structures.

However, out of order of priority, we did write an outline program for the
topological sort.

## What went well

Much to our delight, the experience of verifying in Xr0 is becoming less ad-hoc
and more integrated. We found ourselves writing less hacks to get functions over
the line, and instead had the pleasant surprise, not few times, that we were
able to verify entire functions from "userland" alone.

Our insight also into the nature of the abstract-programming in Xr0 is growing.
As an example, suppose we're modelling a `student` type:

```C
struct student {
	char *name;
	int grade;
};

struct student *
student_create(char *name, int grade) [
	.alloc result;
	result->name = name;
	/* no mention of result->grade because it has no impact on safety */
]{
	struct student *s = malloc(sizeof(struct student));
	s->name = name;
	s->grade = grade;
	return s;
}
```

We may want to write a function that, say, parses a student from a row in a CSV
file, using the following helper functions:

```C
char *
parse_name(char *row) [ .alloc result; ];

int
parse_grade(char *row);
```

The body of the `parse_student` function is straightforward:

```C
struct student *
parse_student(char *row)
{
	char *name = parse_name(row);
	int grade = parse_grade(row);
	return student_create(name, grade);
}
```

But what do we write in its abstract? If we write

```C
struct student *
parse_student(char *row) [
	.alloc result;
];
```

we're kind of stuck because the allocation in `parse_name` takes place before
the allocation for the `student *` returned. So we've settled on writing

```C
struct student *
parse_student(char *row) [
	result = student_create(parse_name(row), parse_grade(row));
];
```

which allows us to express the order in which these allocations take place
properly, and relies on the abstracts of the involved functions to produce the
intended effect in the state.

Because Xr0 is only focused on memory safety, however, we can also write

```C
struct student *
parse_student(char *row) [
	result = student_create(malloc(1), $);
];
```

to express the same thing. This has led us to doubt the felicity of our syntax

```C
.alloc ptr
```

and

```C
.dealloc ptr
```

as well as the `result` keyword. We think that something like

```C
struct student *
parse_student(char *row) [
	return student_create(.alloc(), $);
];
```

would capture the semantic effect of the function better. We also think it may
be useful to allow _abstract-local_ variables in addition to the _body-local_
(usually called _local_) ones.

## Looking forward

The main aim of this iteration (to 30th) will be to find a workable (if
compromised) approach to handling cases with conditional statements. We are also
converting both goals from last week into stretch goals, and maintaining stretch
goal (1.).

Goals:

1. Write test(s) that capture(s) what we think is needed for the conditional
   cases we uncovered
   [[#17](https://todo.sr.ht/~lbnz/xr0/17)]

2. Implement a "low-res" splitting solution to get these to work
   [[#18](https://todo.sr.ht/~lbnz/xr0/18)].

Stretch goals:

1. Verify all but one of the functions that directly invoke malloc (or calloc,
   realloc) and free
   [[#15](https://todo.sr.ht/~lbnz/xr0/15)]

2. Simplify map and array data structures
   [[#13](https://todo.sr.ht/~lbnz/xr0/13)]

3. Write a first-version of the "topological sort" mechanism
   [[#16](https://todo.sr.ht/~lbnz/xr0/16)].
