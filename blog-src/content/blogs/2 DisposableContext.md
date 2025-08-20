---
title: "Hold My Context While I CoC This 🍺🪄"
description: "Need to pass (or get) data downstream in your X++ CoC or framework call without polluting method signatures? Enter Disposable Contexts — your elegant and type-safe solution!"
date: 2025-07-10
draft: false
weight: 2
author: "Anirban Paul"
tags: ["X++", "D365 FinOps", "Extensibility"]
cover: 
    image: "assets/images/DisposableContext.png"
    hidden: false
---

# 🧠 Passing Extra Data in X++ using Disposable Contexts

Ever hit that wall in X++ where you want to pass some extra data downstream — maybe from a CoC extension or a wrapped method — but can’t because the method signature is untouchable? 🤯

Don't worry, you’re not alone. In D365 F&O, it's a common situation. Luckily, there’s a pattern that comes to the rescue like a hero on a white horse: the **Disposable Context** 🐴💾

---

## 🧐 What Problem Are We Solving?

Imagine you’re extending a standard framework class and need to get a value (say, a journal ID) that’s created deep down in the call stack — *without modifying any signatures or breaking extensibility rules.*

You might think of using static variables… but wait — how do you clean them up reliably?

Enter: `System.IDisposable` + `using` + Singleton Context = ✅ Clean, type-safe, and scoped data transfer across classes.

---

## 🧩 How the Pattern Works

Here’s the big idea:

- You create a **singleton context class** that holds your data.
- You **wrap** your call inside a `using` block.
- Downstream code can **get the current context** and write to it.
- When you're done, the context is **disposed** automatically — cleaning up after itself! 🧼

Let’s see a simplified version 👇

---

## 🧱 The Disposable Context Class

{{< highlight xpp >}}
public final class ProdUpdReportFinishedRouteCardJournalContext implements System.IDisposable
{
    private static ProdUpdReportFinishedRouteCardJournalContext instance;
    private ProdJournalId routeCardJournalId;

    private void new()
    {
        if (instance)
            throw error(Error::wrongUseOfFunction(funcName()));
        instance = this;
    }

    public static ProdUpdReportFinishedRouteCardJournalContext construct()
    {
        return new ProdUpdReportFinishedRouteCardJournalContext();
    }

    public static ProdUpdReportFinishedRouteCardJournalContext current()
    {
        return instance;
    }

    public void dispose()
    {
        instance = null;
    }

    public ProdJournalId parmRouteCardJournalId(ProdJournalId _id = routeCardJournalId)
    {
        routeCardJournalId = _id;
        return routeCardJournalId;
    }
}
{{< /highlight >}}

---

## 🏗️ Using the Context in a CoC

Suppose, we want to retrieve `ProdJournalTable.JournalId` from the `usedProdJournalTable()` method and use it in the `updateRouteConsumption()` method to set `parmRouteCardJournalId()`. 

However, since `usedProdJournalTable()` is being called inside (or after) `updateRouteConsumption()`, the `ProdJournalTable` buffer returned from it isn’t maintained as a state and therefore isn’t directly usable.

So, in this case, we can use the Disposable context.

{{< highlight xpp >}}
protected void updateRouteConsumption()
{
    using (ProdUpdReportFinishedRouteCardJournalContext context = ProdUpdReportFinishedRouteCardJournalContext::construct())
    {
        next updateRouteConsumption();

        ProdJournalId routeJournal = context.parmRouteCardJournalId();
        this.parmRouteCardJournalId(routeJournal);
    }
}
{{< /highlight >}}

---

## 🛰️ Downstream Code Sets the Value

{{< highlight xpp >}}
public ProdJournalTable usedProdJournalTable()
{
    ProdJournalTable tableRec = next usedProdJournalTable();
    ProdUpdReportFinishedRouteCardJournalContext ctx = ProdUpdReportFinishedRouteCardJournalContext::current();

    if (tableRec.JournalId && ctx)
    {
        ctx.parmRouteCardJournalId(tableRec.JournalId);
    }

    return tableRec;
}
{{< /highlight >}}

---

## ⚙️ Why Not Use Overloading or Parameters?

Because:
- X++ does **not support method overloading**.
- We can’t change signatures of standard methods in extensions.
- Extensible parameters would be [intrusive](https://docs.microsoft.com/en-us/dynamics365/unified-operations/dev-itpro/extensibility/intrusive-customizations?toc=dynamics365/unified-operations/fin-and-ops/toc.json)

But with:
- A private `new()` constructor 🧱
- Public `construct()` and `current()` 🚪
- Scoped `dispose()` in `using` 🧹

You’ve got a fully controlled, extensible, and clean pattern!

⚠️ Caution:
- Moving state between arbitrary locations using a global variable — which is essentially what the context class is — can introduce subtle logical errors that may be extremely difficult to trace and resolve later. 
-  I recommend restricting its use to cases where the scope and potential side effects are clearly understood and controlled. 
- That's why some consider it as an anti-pattern also.

---

## 🧼 Cleanup is Key: Why IDisposable Matters

Quoting .NET best practices ([docs](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose)):

> The purpose of `IDisposable.Dispose()` is to release unmanaged resources and clean up.

In our case:
- The **resource is the singleton static instance**.
- Calling `dispose()` resets it, preventing data leaks or ghost values. 👻

---

## 🔐 Thread Safety and X++ Specifics

X++ is:
- Single-threaded *per session*
- Static variables are **session-scoped**, not global across users

So if User A and User B both hit the same logic:
- They get **different static instances** of the context 🔐
- Safe from cross-user interference

But still:
> **Don’t use this as a global state store!** Keep it scoped to one logical operation.

---

## 🎯 When to Use Disposable Contexts

✅ When:
- You need to **transfer additional data** downstream in a CoC chain  
- You **can’t modify method signatures**  
- You want **type safety** and **automatic cleanup**

🚫 Avoid:
- Nesting contexts too deep
- Holding context open across unrelated logic
- Using as a long-lived cache

---

## 💬 Final Thoughts

Disposable Contexts are like invisible backpacks 🎒 you carry into a function call — drop things into them downstream, and unpack them when you return!

They’re powerful, clean, and surprisingly elegant for tricky X++ problems involving CoCs, journal creation, or layered customization.

Next time you’re stuck thinking “How do I pass this value without breaking everything?”, remember: **Context is King 👑**

---

## 📚 More Resources  

- [🔗 Implementing IDisposable – Microsoft Learn (.NET)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose)  
  *Covers the purpose and best practices around `System.IDisposable`, which inspired the safe cleanup pattern in X++.*

- [🔗 Disposable Context for X++ – Dynamics Community Blog](https://community.dynamics.com/blogs/post/?postid=fabe0a17-a9f6-4223-9090-0d9643a5ffd0)  
  *The original guide demonstrating 3 approaches to pass data in X++, where Option 3 (Disposable Context) shines as a flexible and extensible solution.*

- [🔗 Disposable context and concurrency](https://community.dynamics.com/forums/thread/details/?threadid=c501ac95-b027-4292-9e68-e70e882aca98#:~:text=In%20any%20case%2C%20for%20F%26O,new%20session%20on%20the%20server)  
  *Clarifies how static variables behave in AX/D365, supporting the safety of singleton patterns like the Disposable Context.*

- [🔗 Garbage Collection and Memory Management in .NET](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)  
  *Explains how cleanup and memory management work under the hood when using `IDisposable` in managed code — relevant since D365 F&O runs on .NET.*
