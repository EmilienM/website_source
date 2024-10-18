---
title: How AI-Assisted is Transforming Software Development
date: 2024-10-18
url: /blog/ai-assisted-coding-tools
author: Emilien Macchi
categories:
  - ai
  - code
cover:
  image: "images/code.jpg"
---

AI-powered coding assistants are revolutionizing the way developers write software.
By providing contextual code suggestions and reducing repetitive tasks, these tools
can significantly increase productivity.
In this post, we'll explore how AI-assisted coding works, its benefits, potential challenges,
and tips for using it effectively in your development workflow. I'll also share
example with two solutions and compare them so it'll help you to make decisions on what tools
to use.

<!--more-->

### The Rise of AI-Assisted Coding

In recent years, machine learning models trained on vast amounts of code have evolved into free or commercial tools,
which integrate seamlessly with popular code editors. These tools use natural language processing and deep learning
to provide real-time code suggestions as you write. Some of these tools also provide a chatbot which can help to explain
some code or even generate it for you based on the questions.

Coding Assistants work analyze the code context and provide auto-suggestions, complete code snippets, or even entire functions.
By learning from repositories (public and/or private), these assistants understand programming patterns and offer relevant solutions.

### Benefits of Using AI Coding Assistants

AI coding assistants can have a huge positive impact on the software development lifecycle. Some of the key benefits include:

- Increased productivity: by suggesting common code snippets, functions, and boilerplate code, AI tools can speed up the development process.
  You spend less time on repetitive tasks and more time solving complex problems.
- Reduced syntax errors: combined with powerful code editors, AI suggestions can minimize syntax errors by offering code that adheres
  to best practices and standards.
- Learning opportunity: developers can learn new APIs, frameworks, and approaches by exploring the suggestions provided by the AI assistant.
- Enhanced focus: instead of switching between the IDE and browser for documentation or Stack Overflow answers, developers can stay focused
  on the code, with instant context-aware help.

### Challenges and Limitations

Like anything with AI, coding assistants offer many benefits, they also come with certain challenges and limitations.

- Code quality concerns: tools suggest solutions that work but are not optimized or could introduce technical debt.
  The responsibility still lies with the developer to review and refine the code. I think that robots aren't (for now) able to replace
  humans to write code that is well architectured and following all best practices.
- Security risks: code assistants might suggest insecure coding patterns or code with vulnerabilities.
  Developers need to remain vigilant, especially when it comes to security-critical code.
  I've experienced a few times where the assistant suggested very dangerous snippets involving the deletion
  of system files for example.
- Over-reliance: developers could become overly reliant on these tools, which could hinder deeper understanding of the code they write.
  This could be problematic for complex or novel problems where the AI may not provide helpful suggestions, especially when you work
  on new products and you need to create new concepts.
- Privacy and IP Issues: some concerns have been raised regarding how AI tools leverage open-source code and whether suggestions violate
  intellectual property or licensing agreements. In this post we'll compare two models where one is open-source.

### Best Practices for Using AI Coding Assistants

To get the most out of AI-powered coding tools, it’s important to follow a few best practices:

- Use AI as a complement, not a crutch: AI should assist in the development process but not replace thoughtful coding.
  Use the suggestions as a foundation, but always review and customize the code to fit your specific use case.
- Understand the code: don't blindly accept suggestions. Make sure you understand the code being suggested and test it thoroughly.
- Focus on complex problem solving: let AI handle the repetitive or boilerplate tasks so you can spend more time on complex logic and design decisions.
- Stay up to date on best practices: AI tools will continue to improve, but developers need to stay informed about programming best practices,
  especially around security, performance, and maintainability. Keep learning, take trainings, review other's code and also keep up with
  the models you've been using for code assistant. New releases often offer interesting features.

### My toolbox

I've been playing with code generation for quite a while now and I want to share a couple of tools that I use:

- [Github Copilot](https://github.com/features/copilot): commercial solution for AI-powered coding assistant that provides
  real-time code suggestions directly within your code editor.
- [Granite Code](https://www.ibm.com/granite/playground/code/): open-source models that are decoder-only models designed for
  code generative tasks, trained with code written in 116 programming languages.

I won't explain how to setup your IDE to use these tools, I suggest to read their official documentation.
Note that to run Granite models locally, you'll certainly need a GPU if you want the code assistant to really be usable.

### Before we start

This is not a detailed comparaison between the two models but rather a quick example on that these tools can offer so you get an idea if you haven't tried them yet.

Benchmarking the models is not the goal here.

### Github Copilot

Copilot offers two extensions:

- Github Copilot: provides inline coding suggestions as you type.
- GitHub Copilot Chat: provides conversational AI assistance.

Let's use the chat first:

![Python script generated by Copilot](/images/code-ai-python-copilot.png)

The generated snippet just works.

Now let's see if we can print the Python version directly by
commenting a function that doesn't exist yet.

![Code generation with Copilot](/images/code-ai-python-copilot.gif)

Here is another example involving basic mathematics:

![Another Python script generated by Copilot](/images/code-ai-python2-copilot.png)

Now let's ask Copilot to generate unit tests for that script (using the "Generate tests button"):

![Unit tests generated by Copilot](/images/code-ai-tests-copilot.png)

### Granite

Now let's play with the Granite 8B code model.
I've installed the [Continue](https://www.continue.dev) extension which
provides both inline coding suggestions (if the model permits) and
a chat.

As previously done, let's first talk to the chat to initate the script.

![Python script generated by Granite](/images/code-ai-python-granite.png)

The result is in doing the job with almost the same result.
You'll notice that it didn't create a function but rather printed directly what we asked for.

Let's see if it can find out what version of Python is running:

![Code generation with Copilot](/images/code-ai-python-copilot.gif)

And the other example:

![Another Python script generated by Granite](/images/code-ai-python2-granite.png)

Unit tests:

![Unit tests generated by Copilot](/images/code-ai-tests-granite.png)

If you noticed, the test will fail. It missed `5` which is in the first list and is odd.
The test wasn't as extended as Copilot suggested and in this case, wasn't working.


### My take on the models

Again the examples were very basic (on purpose) so you can have a slight overview on what to expect.
The results are pretty good in both models and can really help to increase productivity.

### Conclusion

AI-powered tools are transforming the way we write software by making coding more efficient, accessible, and collaborative.
However, they're not a silver bullet. Developers should use these tools thoughtfully, balancing the convenience of AI assistance
with the need to maintain control and understanding of the code they write.

By adopting AI-assisted coding into your workflow, you can boost productivity and learn new approaches, but don't forget that good
coding practices and creativity will always remain central to successful software development.

Now it's up to you to test the tools, find models that fit your needs (and company policy!) which will hopefully help you.

Note: the image of this post was generated by Llama3 using the Facebook chat. The prompt was "generate an image of a developer assisted by a robot".
