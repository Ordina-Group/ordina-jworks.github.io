---
layout: post
authors: [oumaima_zerouali]
title: "Uncle Bob's Clean Code & The Clean Coder: From a junior's point of view"
image: /img/2024-10-25-clean-code-and-clean-coder/banner.png
tags: [ Robert C Martin, Clean code, Clean coder]
category: Culture
comments: true
---
> In coding, clean code is like the Krabby Patty secret formula—it’s all about the right ingredients for success!

{:.no_toc}
- TOC
{:toc}

## Introduction
When I first started as a junior developer, I was focused on one thing: making my code work. 
I was focused on learning the basics, solving bugs, and understanding how all the pieces fit together. 
In the rush to learn everything; I often overlooked the fundamentals of writing clean, maintainable code.
Writing tests, or thinking about readability, or something as basic as meaningful variable names all felt secondary when I was trying to get the program to work.

However, in my effort to learn and grow, a colleague recommended that I read Uncle Bob's Clean Code and The Clean Coder. 
These books taught me the value of clean code. 
And as a junior developer, I want to share some of the key lessons that have changed the way I approach my work.

## Clean code
<img alt="mentoring" src="{{ '/img/2024-10-25-clean-code-and-clean-coder/clean-code.png' | prepend: site.baseurl }}"  style="float: right !important; max-width: 180px;">

Published in 2008, Uncle Bob’s Clean Code refers to a set of guidelines and practices aimed at making software easier to understand and maintain.

I'll be discussing three principles from Uncle Bob's Clean Code that I've implemented the most in my work. 
While there are many valuable principles outlined in the book, these particular ones form the basis in my opinion. 
Additionally, I’ll explore two concepts that I found especially compelling: Test-Driven Development (TDD) and the Boy Scout Rule.

### Basic Clean Code Principles
#### 1. Meaningful Variable and Function Names
Using descriptive names helps convey the purpose of variables and functions, making the code easier to understand.

Example: Instead of naming a variable `d` (elapsed time in days), use `elapsedTimeInDays` to clearly indicate its purpose. 
Or instead of a function name called `calc()`, use `calculateElapsedTime()`.
```java
    int elapsedTimeInDays;
    
    int calculateElapsedTime(int startDay, int endDay) {
        return endDay - startDay;
    }
```

#### 2. Keep Functions and Methods Short
In the book Uncle Bob tells us the first rule of functions is that they should be small.
The second rule? They should be even smaller.
This principle encourages developers to break down complex operations into smaller, focused functions that perform a single task.

Example: Instead of a long method that does multiple things, you could break it down into smaller methods.
```java
    // Long method that does multiple things
    void processOrder(Order order) {
        // Validate the order
        if (order == null || order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Invalid order");
        }
    
        // Calculate total price
        double total = 0;
        for (Item item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        order.setTotalPrice(total);
    
        // Send confirmation email
        EmailService emailService = new EmailService();
        emailService.sendOrderConfirmation(order.getCustomerEmail(), order);
    }

    
    // Short method that will call other methods
    void processOrder(Order order) {
        validateOrder(order);
        calculateTotal(order);
        sendConfirmation(order);
    }
```

#### 3. DRY (Don't Repeat Yourself) Principle
As Uncle Bob states, "Duplication may be the root of all evil in software."
The DRY principle emphasizes the importance of reducing duplication in your code to reduce the potential for errors.
When the same code appears in multiple places, any changes require updates in every location, increasing the risk of inconsistencies and errors.

Example: Instead of having those 2 identical methods we could make a processOrder and call it for both Standard and Express.
```java
    // Duplicated
    void processStandardOrder(Order order) {
        validateOrder(order);
        calculateTotal(order);
        sendConfirmation(order);
    }

    void processExpressOrder(Order order) {
        validateOrder(order);
        calculateTotal(order);
        sendConfirmation(order);
    }
    
    // DRY
    void processOrder(Order order) {
        validateOrder(order);
        calculateTotal(order);
        sendConfirmation(order);
    }
```

### TDD (Test-Driven Development)
One of the concepts that really stood out to me from Uncle Bob’s teachings is Test-Driven Development (TDD). 
As a junior developer, I was used to writing code first and then testing it afterward, but TDD flips that process around. 
Uncle Bob advocates for writing tests before you even write a single line of production code. 
At first, this idea seemed a bit strange to me, why would I write tests for something that doesn’t even exist yet?

Writing tests beforehand forces you to clarify exactly what the code is supposed to do. 
TDD also creates a constant feedback loop. 
Every time I wrote a piece of code, I knew whether it worked because the tests either passed or failed. 
This continuous feedback was helpful as I iterated through improvements and fixes. 
It also gave me the confidence that once the code worked, it was thoroughly tested and less likely to break in the future.

### The Boy Scout Rule
In my opinion the coolest concept I encountered in Clean Code was the Boy Scout rule: "Always leave the campground cleaner than you found it." 
This simple idea really struck a chord with me. 
It means that every time you touch a piece of code, whether you're fixing a bug or adding a new feature, you should make an effort to improve it, even in small ways. 
You don’t have to rewrite the whole thing, but maybe rename a confusing variable or break down a large function.

When I first started applying this principle, it felt awesome. 
Not only did I feel like I was leaving the codebase in a better state, but it also forced me to slow down and really understand what the existing code was doing. 
In the past, I might have just rushed through a change to get it working, but taking the time to improve the code helped me grasp the logic behind it more deeply.

## The Clean Coder
<img alt="mentoring" src="{{ '/img/2024-10-25-clean-code-and-clean-coder/the-clean-coder.png' | prepend: site.baseurl }}"  style="float: left !important; max-width: 180px; margin-right: 20px;">

Published in 2011, Uncle Bob's The Clean Coder takes a deeper dive into the mindset and principles that underpin professional software development. 
While Clean Code focuses on the technical aspects of writing code, The Clean Coder emphasizes the importance of professionalism and ethical responsibility in the craft of coding. 
Uncle Bob encourages developers to adopt a disciplined approach to their work, stressing that being a clean coder means not only producing high-quality code but also valuing communication, collaboration, and continual learning.

In this book, Uncle Bob shares insights on how to approach various aspects of software development, from managing time and prioritizing tasks to dealing with difficult situations and making decisions. 
He argues that professionalism is key to building a successful career in software development, and he provides practical advice for navigating the challenges developers face daily.

### Professionalism
In The Clean Coder, Uncle Bob makes a clear distinction between being just a programmer and being a professional programmer. 
He argues that professionalism isn’t just about writing code, it’s a responsibility you take on. 
When you call yourself a professional programmer, you are committing to a certain standard of behavior and quality in your work.

Uncle Bob emphasizes that being a professional means more than just delivering code that works. 
It involves taking ownership of your work, ensuring that what you deliver is high-quality, maintainable, and on time. 
A professional programmer understands the importance of meeting deadlines and takes responsibility for their code, even when things go wrong. 
They communicate clearly with their team, are open to feedback, and constantly strive to improve their skills.

### Practicing
Uncle Bob tells us, that as developers, we should always be working to improve our skills. 
He encourages us to refine our abilities through regular coding exercises, side projects, or collaborative coding sessions with others. 
Uncle Bob’s suggests that if you work 40 hours a week, you should be spending an additional 20 hours on self-improvement. 
Now, don’t take that literally, but understand the essence of what he’s saying. 
It’s not necessarily about the amount of time; it’s about making learning a priority.
Whether it’s reading a book, working on a side project, doing coding katas, or even just experimenting with new technologies, the key is to dedicate time to growing as a developer—or, as Uncle Bob says, to becoming a professional programmer.

Since I've started doing it, I’ve found that the more I practice outside of work, the better I perform when I’m on the job. 
Working on side projects, for instance, has exposed me to different challenges and has helped me think more creatively when solving problems at work. 
Reading technical books has also broadened my understanding of concepts I might not have encountered in my day-to-day tasks.

### Mentoring and Apprenticeship
In the book, Uncle Bob highlights the significance of mentorship in a developer's journey. 
He argues that learning from experienced programmers can accelerate growth and help you navigate the complexities of software development more effectively. 
A good mentor not only shares technical knowledge but also offers insights into professional behavior, decision-making, and the overall mindset needed to succeed in this field.

From my perspective, having a solid mentor has been a game-changer for me as a programmer. 
I've been fortunate to work alongside colleagues who are not only skilled but also willing to share their expertise and provide guidance.
<img alt="mentoring" src="{{ '/img/2024-10-25-clean-code-and-clean-coder/mentoring.png' | prepend: site.baseurl }}"  style="margin-left:250px; max-width: 400px;">

## Conclusion
As I continue my journey as a developer, I realize that I’m learning something new every day. 
Uncle Bob’s Clean Code and The Clean Coder have played a huge role in helping me understand what coding is really about. 
These books have shifted my perspective, guiding me toward becoming a more thoughtful and responsible developer.

But as much as these books have helped, I couldn’t have made the progress I’ve made without the support of my colleagues. 
They’ve been there to explain concepts when I didn’t quite understand, debate the principles of clean code with me, and always push for the best possible solutions. 
Their willingness to share knowledge and their commitment to excellence have been just as influential in my growth.

Right now, I’m diving into Clean Architecture, and I’m excited to explore how these principles intertwine with what I’ve already learned. 
Stay tuned—who knows? I might just write a post about it in the future!

In the world of software development, Uncle Bob truly is king, but I’ve learned that having great colleagues by your side is just as valuable. 
Their insights and dedication inspire me to keep striving for excellence.