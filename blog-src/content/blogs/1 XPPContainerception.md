---
title: "🥡 How to Pack a Container Like a Bengali Mom Packs a Lunchbox"
date: 2025-07-09
draft: false
weight: 1
author: "Anirban Paul"
tags: ["X++", "D365 FinOps", "Tips", "Data Structures"]
cover: 
    image: "assets/images/XPPContainerception.png"
    hidden: false
---

🧊 **Ever wanted to insert one container inside another in X++?**  
Whether you're working with temporary in-memory structures or building payloads for integrations, this trick can help you dynamically group rows into containers — just like a 2D array!

Let’s dive in. 💻

---

## 🚀 The Problem

You want to loop through a dataset (like IDs and names), create a container for each row, and **nest those containers into one master container**.

Sounds simple — but you need to do it **the right way** to avoid mistakes like flattening your data.

---

## 🧪 Solution 1: Using `+=` (Append Operator)

This is the **cleanest and most common** way to insert a container into another container.

```x++
static void InsertUsingPlusEquals(Args _args)
{
    container parentCon;
    container childCon;
    int       id;
    str       name;

    for (int i = 1; i <= 5; i++)
    {
        id = i;
        name = strFmt("Name_%1", i);

        childCon = conNull();     // Clear before reuse!
        childCon += id;
        childCon += name;

        parentCon += [childCon];  // Correct: Wrap in [] to insert as 1 item
    }

    // (Optional) we print to verify
    for (int j = 1; j <= conLen(parentCon); j++)
    {
        container row = conPeek(parentCon, j);
        info(strFmt("Row %1 - ID: %2, Name: %3", j, conPeek(row, 1), conPeek(row, 2)));
    }
}
```

### 🔥 Key Insight:
- **`+= [childCon]`** adds the whole child container as a **single element** in `parentCon`.
- Without square brackets (`[]`), it **unpacks** the child and appends individual elements. ⚠️

---

## 🧪 Solution 2: Using `conIns()` (Insert at Specific Index)

If you want more control — like inserting at the beginning or a specific position — `conIns()` is your friend.

```x++
static void InsertUsingConIns(Args _args)
{
    container parentCon;
    container childCon;
    int       id;
    str       name;

    for (int i = 1; i <= 5; i++)
    {
        id = i;
        name = strFmt("Name_%1", i);

        childCon = conNull();
        childCon += id;
        childCon += name;

        // Insert at the end using length + 1
        parentCon = conIns(parentCon, [childCon], conLen(parentCon) + 1);
    }

    // (Optional) we print to verify
    for (int j = 1; j <= conLen(parentCon); j++)
    {
        container row = conPeek(parentCon, j);
        info(strFmt("Row %1 ➡ ID: %2, Name: %3", j, conPeek(row, 1), conPeek(row, 2)));
    }
}
```

### 🧠 Why Use `conIns()`?
- Great when order matters!
- Insert at **start**, **middle**, or **end** by changing the index:
  ```x++
  parentCon = conIns(parentCon, [childCon], 1); // Insert at beginning
  ```

---

## 🛠️ Summary

| Feature             | `+=`                   | `conIns()`                             |
|---------------------|------------------------|----------------------------------------|
| Append at end       | ✅ Simple and readable | ✅ More verbose                       |
| Insert at position  | ❌ Not possible        | ✅ Can insert anywhere                |
| Performance         | ⚡ Fast                | ⚡ Fast (but slightly more overhead)  |
| Readability         | ✅ Clean               | 🛠️ Useful for ordered inserts         |

---

## 💬 Final Thoughts

This is a go-to technique in **D365 FinOps** or **AX** development when:
- Creating nested in-memory records
- Building payloads for AIF/JSON/XML
- Simulating rows like a mini temp table

> 🧰 **Pro Tip**: Wrap your logic in a helper method or class to reuse it across multiple jobs or classes!

Happy coding, devs! 🚀🧑‍💻  
Let your containers contain with confidence!

---

## 📚 More Resources

- 🔗 [Insert Containers into Container in X++ (Original Reference)](https://alexvoy.blogspot.com/2010/02/insert-containers-into-container.html)  
  A classic community blog post showing the basics of nesting containers in X++.

- 📘 [Microsoft Learn – Data Types: Container](https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/dev-ref/xpp-container-run-time-functions)  
  The official documentation for understanding containers, `conPeek()`, `conIns()`, and more in X++.
