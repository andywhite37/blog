---
title: "My First Post"
date: 2019-10-31T21:11:53-06:00
draft: false
---

# H1

## H2

### H3

#### H4

##### H5

###### H6

Some content

:smile:

```ocaml
module List = {
  type t('a) = list('a);

  let map = Belt.List.map;

  module Functor: FUNCTOR with type t('a) = t('a) = {
    type nonrec t('a) = t('a);
    let map = map;
  };
};
```