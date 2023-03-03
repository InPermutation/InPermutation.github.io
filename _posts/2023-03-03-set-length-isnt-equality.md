---
layout: post
title: "Equal set length doesn't imply set equality"
date: "Fri, 03 Mar 2023 17:14:08-0400"
---
On one of my projects,
I happened to glance at a legacy function that was supposed to check
if the user had permission to add or remove some `Doohickeys` from a `Widget`.
But it wasn't actually doing that:

```csharp
record Doohickey
(
    string Id,
    string Name
);

record Widget
(
    string Id,
    string Name,
    ImmutableHashSet<Doohickey> Doohickeys
);

class WidgetPermissionCheck : UserPermissionCheck
{
    public void AssertOnBeforeSave(Widget old, Widget @new)
    {
        Assert(User, Permissions.CanWriteWidgets);

        if (old.Doohickeys.Count > @new.Doohickeys.Count)
        {
            Assert(User, Permissions.CanRemoveDoohickeys);
        }

        if (old.Doohickeys.Count < @new.Doohickeys.Count)
        {
            Assert(User, Permissions.CanAddDoohickeys);
        }
    }
}
```

What went wrong? There is a vulnerability when the user adds and removes `Doohickeys` at the same time.

If you add more than you remove, you bypass the `CanRemoveDoohickeys` check.

If you remove more than you add, you bypass the `CanAddDoohickeys` check.

If you add and remove the same number, you bypass both of these checks!


```csharp
        if (old.Doohickeys.Except(@new.Doohickeys).Any())
        {
            Assert(User, Permissions.CanRemoveDoohickeys);
        }

        if (@new.Doohickeys.Except(old.Doohickeys).Any())
        {
            Assert(User, Permissions.CanAddDoohickeys);
        }
```

You can't just check the count, you have to actually compare which `Doohickey`s were added or removed.

Fortunately, on this project,
the permission check was actually superfluous,
as all users with `CanWriteWidgets` were also entrusted with both
`CanRemoveDoohickeys` and `CanAddDoohickeys` permissions.
But this is a scary risk to be aware of.
