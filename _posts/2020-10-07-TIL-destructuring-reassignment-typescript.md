---
layout: post
title: Typescript destructuring re-assignment
tags: [til, typescript]
categories: til typescript
date: 2020-10-07 09:37:00 +0200
toc: true
---

Destructuring needs to be either after a `let`, `const` or `var` declaration **OR** it needs to be **in an expression context** to distinguish it from a block statement.

```tsx
let variable: number

if (true) {
	({ bla: variable } = someFunction());
} else {
	({ bla: variable } = someOtherFunction());
}
```