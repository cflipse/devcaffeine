---
layout: post
title: "Constants are global"
---

New developers are socialized early on with the idea that global variables
are a thing to be avoided.  However, many developers, both new and experienced
have no problem burying class method calls deep within the methods of a service
object.  I contend that there is no effective difference between these methods:

```
def foo
  users = User.active
  ...
end

def bar
  users  = $users.active
  ...
end
```

