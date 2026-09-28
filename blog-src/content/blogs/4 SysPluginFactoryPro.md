---
title: "🤖 Before Copilot, There Were Plugins: The SysPlugin Story in D365FO"
date: 2026-09-28
draft: false
weight: 4
author: "Anirban Paul"
tags: ["X++", "D365 FinOps", "Design Pattern"]
cover: 
    image: "assets/images/SysPluginFactory.png"
    hidden: false
---

> “Switch-cases are dead. Long live plugins!” — A sleep-deprived developer after finally getting `SysPluginFactory` to work

---

## ⚡ SysPlugin Framework in D365 F&O – The Secret Sauce Behind Pluggable Code 🍔

If you thought **SysExtension** was cool (goodbye, massive `switch-case` blocks 👋), wait till you meet its cousin — the **SysPlugin framework**. Think of SysPlugin as SysExtension but with *superpowers* ⚡. Instead of just resolving classes dynamically, it lets you decide **which plugin to use at runtime based on context**. Yup, smarter decisions, less boilerplate, more flexibility. 🚀

---

## 🎯 What is SysPlugin Framework?

At its core, `SysPluginFactory` is a **Managed Extensibility Framework (MEF)-style plugin discovery system** in D365FO.  

Think of it like a **universal adapter** 🔌 – you define a contract (interface/abstract class), register your plugins (implementations), and the framework picks the right one when needed.

---

## 🧩 Where Can You Use It?

- Replace ugly `switch` or `if-else` ladders 🚫
- Inject custom business rules without modifying base code 🛠️
- Plug in multiple implementations based on conditions 🌍
- Make your code **future-proof and extensible** 🔮

---

## 🧠 The Theory (aka MEF in X++ clothes)

Here’s what happens under the hood:

1. **Define an interface** (`SysIMailer` in our example).  
2. **Export your implementation** with attributes:  
   - `[ExportAttribute]` → tells D365FO “this class implements something.”  
   - `[ExportMetadataAttribute]` → attaches metadata like IDs or types for filtering.  
3. **Use SysPluginFactory** to discover instances:
   - `SysPluginFactory::Instance()` → get one implementation by metadata.  
   - `SysPluginFactory::Instances()` → get all implementations of an interface.  
4. The framework wires everything together dynamically at runtime.  

So unlike `SysExtension` (which matches **one subclass per key**), `SysPlugin` is designed to **handle multiple plugins and let you pick/filter at runtime.**  

---

## 📦 Example: SysMailer Framework (Emails Without Tears)

Let’s see how Microsoft has developed an extensible mailer system where different providers (SMTP, Microsoft Graph, etc.) can be plugged in **without touching the factory code.**

---

### 1. Define the Interfaces

We start with a core contract:  

```x++
[Microsoft.Dynamics.AX.Platform.Extensibility.ExportInterfaceAttribute]
public interface SysIMailer
{
    SysMailerId getId();
    SysMailerDescription getDescription();
}
```

Then extend it for interactive and non-interactive senders:

```x++

public interface SysIMailerInteractive extends SysIMailer
{
    boolean sendInteractive(System.Net.Mail.MailMessage _message);
}

public interface SysIMailerNonInteractive extends SysIMailer
{
    boolean sendNonInteractive(System.Net.Mail.MailMessage _message);
}
```

Boom. Clear contracts. No one cares how the mail is sent — only that it can be.


### 2. The Factory (SysMailerFactory)

Here’s where SysPluginFactory does the magic:

```x++
public static class SysMailerFactory
{
    public static SysIMailer getMailer(SysMailerId _mailerId)
    {
        SysPluginMetadataCollection metadataCollection = new SysPluginMetadataCollection();
        metadataCollection.SetManagedValue(extendedTypeStr(SysMailerId), _mailerId);

        return SysPluginFactory::Instance(
            identifierStr(Dynamics.AX.Application),
            classStr(SysIMailer),
            metadataCollection
        );
    }

    public static Map getMailers()
    {
        var mailers = SysPluginFactory::Instances(
            identifierStr(Dynamics.AX.Application),
            classStr(SysIMailer),
            new SysPluginMetadataCollection()
        );

        var mailerMap = new Map(Types::String, Types::Class);

        for (var i = 1; i <= mailers.lastIndex(); i++)
        {
            SysIMailer mailer = mailers.value(i);
            Debug::assert(!mailerMap.exists(mailer.getId()));
            mailerMap.insert(mailer.getId(), mailer);
        }

        return mailerMap;
    }
}

```

💡 Notice: No switch-case, no if (id == “SMTP”), nothing.
Just ask SysPluginFactory nicely, and it hands you the correct plugin. 🪄


### 3. Register Providers (Plugins)

Each mailer registers itself with attributes:

```x++
#define.SysMailerGraph_ID('Graph')

[ExportAttribute(identifierStr(Dynamics.AX.Application.SysIMailer)),
 ExportMetadataAttribute(extendedTypeStr(SysMailerId), #SysMailerGraph_ID)]
public final class SysMailerGraph extends SysMailerThrottled implements SysIMailerInteractive
{
    public SysMailerId getId() { return #SysMailerGraph_ID; }
    public SysMailerDescription getDescription() { return "@ApplicationFoundation:EmailProviderGraphDescription"; }

    public boolean sendInteractive(System.Net.Mail.MailMessage _message)
    {
        // Graph API magic ✨
        return this.sendMessageThrottled(_message, true, newGuid());
    }

    public boolean sendNonInteractive(System.Net.Mail.MailMessage _message)
    {
        return this.sendMessageThrottled(_message, false, newGuid());
    }
}

```

And another:

```x++
#define.SysMailerSMTP_ID('SMTP')

[ExportAttribute(identifierStr(Dynamics.AX.Application.SysIMailer)),
 ExportMetadataAttribute(extendedTypeStr(SysMailerId), #SysMailerSMTP_ID)]
public class SysMailerSMTP extends SysMailerThrottled implements SysIMailerInteractive
{
    public SysMailerId getId() { return #SysMailerSMTP_ID; }
    public SysMailerDescription getDescription() { return "@ApplicationFoundation:EmailProviderSMTPDescription"; }

    public boolean sendInteractive(System.Net.Mail.MailMessage _message)
    {
        // SMTP magic 📨
        return this.sendMessageThrottled(_message, true, newGuid());
    }

    public boolean sendNonInteractive(System.Net.Mail.MailMessage _message)
    {
        return this.sendMessageThrottled(_message, false, newGuid());
    }
}

```

That’s it. Two providers, no changes in the factory. Plug-and-play extensibility. 🎉


### 4. Using It

```x++
public static void sendEmail()
{
    System.Net.Mail.MailMessage msg = new System.Net.Mail.MailMessage();
    msg.set_Subject("Hello from SysPlugin!");
    msg.set_Body("This is a plugin-powered email 🚀");

    SysIMailer mailer = SysMailerFactory::getMailer('SMTP');
    if (mailer)
    {
        mailer.sendNonInteractive(msg);
    }
}

```
🎯 The calling code doesn’t know if it’s SMTP, Graph, or Pigeon Post.
It just knows: “I asked for a mailer. I got one. It worked.”

---

## 🧩 Design Patterns & SOLID Principles

This framework is a textbook example of good software design:

Factory Pattern → SysMailerFactory abstracts away instantiation.

Dependency Inversion Principle (D in SOLID) → High-level code depends on abstractions (SysIMailer) not implementations.

Open/Closed Principle (O in SOLID) → Add new providers (Graph, SMTP, SendGrid) without modifying factory logic.

In short: OCP + DIP = happy developers. 💡

---

## ✨🤖 From Process Automation to Copilot: Plugins Are Everywhere

Now here’s where **SysPlugin gets really interesting**.

It isn’t just a framework you can use for your own `SysMailer`, payment providers, or other extensible components. Microsoft uses the **SysPlugin pattern inside major D365FO frameworks too**.

One great example is the **Process Automation Framework**.

The framework uses plugin interfaces extensively — often passing the **process type name as metadata** so that the correct implementation is discovered and invoked for that particular process. In other words, the framework doesn't need a giant `switch` saying *“if this is Vendor Payment Proposal, call this class; if this is Invoice Posting, call that class…”*.

Instead:

**Export → Metadata → Discover → Instantiate → Execute.** 🔌✨

That same pattern shows up in areas such as process type registration, occurrence handling, parameter management, and other Process Automation extension points.

👉 [🔗 Process Automation Framework Development](https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/process-automation/process-automation-framework)  
👉 [🔗 Process Automation Type Registration](https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/process-automation/type-registration)

And then comes the really fun part... 🤖

### 🚀 Plugins Meet AI

D365FO is increasingly exposing its business logic to **Copilot and AI agents**.

With **Copilot client plugins**, developers can expose X++ business logic as actions that Copilot can invoke through natural-language prompts. Microsoft describes these plugins as a way to turn existing application operations and business logic into actions that users can trigger from the Copilot experience.

And the newer **AI tools** direction takes this even further: business logic can be exposed as headless operations that AI agents and copilots can invoke without depending on a specific F&O client experience. 🤯

So the bigger picture looks something like this:

```text
                    ┌───────────────────┐
                    │   Copilot / AI    │
                    └─────────┬─────────┘
                              │
                         Natural Language
                              │
                    ┌─────────▼─────────┐
                    │  AI / Client      │
                    │     Plugin        │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │   D365FO Business │
                    │      Logic        │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │   SysPlugin /     │
                    │  Metadata-based   │
                    │    Discovery      │
                    └─────────┬─────────┘
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
        Implementation    Implementation    Implementation
             A                 B                 C
```

And that is why understanding SysPlugin isn't just about learning another factory pattern.

The same fundamental idea — define a contract, register implementations, attach metadata, and let the framework discover the right implementation — appears in extensible D365FO frameworks today and fits naturally into the direction of automation, Copilot, and agentic ERP. 🚀

---

## ⚡ Two frameworks, one purpose
 
X++ factory methods use the strategy pattern: a base class (or interface) defines the contract, subclasses provide the variants, and a factory picks the right variant at runtime. **SysExtension** and **SysPlugin** are the two extension frameworks a factory can use to find that variant.

### Comparison
 
| | SysExtension | SysPlugin |
|---|---|---|
| Built on | X++ custom attributes | Managed Extensibility Framework (MEF) |
| Registration | Decorate the subclass with a custom attribute | `ExportMetadataAttribute` (the variant key) plus `ExportAttribute` (makes it discoverable) |
| Key type | Strongly typed attribute (enums, including extensible enums, work seamlessly) | String-based metadata |
| Matching | Factory searches the class hierarchy for a subclass whose attributes match the parameters passed in | Consumer discovers exports of a contract and filters on their metadata |
| Instance creation | Reflection, with optional singleton support (helps with stateless subclasses created repeatedly) | Handled by the consuming framework |
| Reachable from non-X++ code | No | Yes, because it is MEF-based |

### Which one to use
 
- **Extending Copilot client actions, process automation, or any other SysPlugin-based framework:** use SysPlugin. It is the mechanism the framework consumes.
- **Extending a SysExtension-based factory:** use the attribute the factory searches for.
- **Writing your own factory for X++-only variants:** SysExtension is the usual fit. It gives you typed keys, extensible enum support, and optional singletons.
- **Writing your own factory where variants must be discoverable from non-X++ code, or the key is naturally a string or metadata value:** SysPlugin fits better.

### Things to watch
 
- SysPlugin keys are strings, so a typo compiles fine and fails at runtime. Use `classStr()`, `formStr()` and `identifierStr()` where possible instead of literals.
- SysPlugin is a discovery and registration mechanism. It is not a full dependency-injection container, and it does not do constructor injection or lifetime management.
- SysExtension is not a "factory replacement". Factory methods use it to find the correct subclass.
---

## 📚 More Resources

- [🔗 Microsoft Docs – SysPlugin Framework](https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/extensibility/register-subclass-factory-methods)  
- [🔗 Ievgen’s Blog – Plugins in D365FO](https://ievgensaxblog.wordpress.com/tag/syspluginfactory/)  
- [🔗 Dax Tech Solutions – Code Extensions with Plugins](http://daxtechsolutions.blogspot.com/2018/08/d365ax7-code-extensions-using-plugins.html) 
- [🔗 Create X++ Client Plugins for Copilot Studio in Dynamics 365 F&O By Ariste](https://ariste.info/2025/09/create-client-plugins-copilot-studio-d365-fo/)
---

## 🎯 Final Thoughts: Pluggin’ It In

The SysPlugin framework is like the power strip of D365FO:

You don’t care which device you plug in.

You just know it’ll work, as long as it fits the interface. ⚡

So next time you need extensibility with multiple implementations, don’t reach for a switch-case.
Just plug it in. 🔌

Happy pluggin’, fellow warriors! 🧙‍♂️📬