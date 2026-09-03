# 01 — Mindset & Approach

[▶ 0:00](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=0s)

The first 28 minutes have no code. They set up the whole series. Read this once, then move on.

---

## What this video is

It is a companion to [A Tour of Go](https://go.dev/tour/welcome/1). It is not a replacement for it.

The author copies each Tour example into a local editor, runs it, and explains it in his own words.

One thing makes it different from other tutorials. After each concept, he opens a real production codebase (fider, incubator-answer) and shows the same concept being used there. The goal is to close the gap between learning a concept and seeing it in real code. Normally that gap takes years. Here it takes seconds.

## What this video is not

It is not a full tutorial that covers every corner of the language.

It does not cover installing Go or setting up your editor. That is considered out of scope. You are expected to figure it out.

It is scoped to what a **backend developer** needs. It is not aimed at library authors.

## Prerequisites

You need basic programming knowledge. You should know what variables, arrays, and loops are.

Experience in another language helps. It is not required.

---

## The recommended workflow

1. Stop now and do the entire Tour of Go yourself. Read the left pane. Run the code in the right pane. It takes 2–3 hours in one sitting.
2. Do not worry about understanding it. The goal of the first pass is to get a feel for the syntax.
3. Then watch the video or read these notes for the explanation and the real-world context.

> "Before you understand the concept you should have a feel of the syntax."

---

## The main idea: programming is pattern matching

Being a good programmer has little to do with talent. It comes down to **pattern matching**. That means reaching for the right tool at the right moment without thinking about it.

Here is an example. You see a comma-separated string and you need an array. You immediately think of `split`. It does not matter whether you are in Python, Java, Go, or JavaScript. You do not reason it out. That is pattern matching.

Pattern matching improves with volume. The more code you write and read, the better it gets. Eventually it becomes muscle memory.

This changes how you should learn. Understanding should not be your priority. Consistency and volume should be. Understanding comes later on its own.

---

## Two pieces of advice

### 1. Don't try to understand everything

The worst thing you can do with a new language is demand that you understand all of it right away. It only makes you feel intimidated. Feeling intimidated is what makes people quit.

If something does not click in this video, it will click later. It will click while reading standard library code. Or while reading open source projects. Or while building your own projects.

Trust the process. Keep writing. Keep reading.

### 2. Some Go concepts can wait

Tutorials spend a lot of time on interfaces, error handling, concurrency (fan-in, fan-out, mutexes, select, channel directions), and reflection.

For an application developer, these are about **20% of what you actually use**.

Interfaces are the exception. You will use those a lot.

Advanced concurrency patterns are different. You can work for years without writing fan-out or fan-in yourself. You will use them indirectly through libraries.

Testing matters, but it belongs to the project-building phase. Not the basics phase.

Just know these concepts exist. Know roughly when they are needed. Then move on. Treating them as blockers while learning the basics is what makes people give up on Go and call it a complicated systems language.

---

## When reading the real-world code examples

The video jumps into fider and incubator-answer often. When it does, look only at the lines being pointed at.

Do not read the surrounding code. It will only intimidate you. Reading whole codebases comes later in the series.

---

## What success looks like

[▶ 5:51:58](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=21118s)

> "My goal with this video was to get you to 10%."

The video openly admits it skips things. It skips reflection, directional channel types, and anything the Tour of Go does not cover.

The remaining 60% comes from the rest of the series. That covers Effective Go, the modules reference, the language spec, reading standard library source, reading production open source backends, and finally building projects.

---

**Next:** [02 — Modules & project setup](02-modules-and-project-setup.md)
