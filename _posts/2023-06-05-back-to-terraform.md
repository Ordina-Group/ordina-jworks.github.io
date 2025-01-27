---
layout: post
authors: [pieter_vincken, yannick_bontemps]
title: 'Back to Terraform'
image: /img/2023-06-05-back-to-terraform/header.png
tags: [cloud, automation, cicd, terraform, iac]
category: Cloud
comments: true
---

> “Fail fast” does not imply lack of commitment to a mission or goal, but on the contrary, indicates a willingness to experiment in the process, learn quickly from the results, and make adjustments to better achieve an enhanced customer experience.

- [Introduction](#introduction)
- [What is Terraform?](#what-is-terraform)
- [What is Pulumi?](#what-is-pulumi)
- [What is Terragrunt?](#what-is-terragrunt)
- [Developer adoption, excluding the batteries: Pulumi](#developer-adoption-excluding-the-batteries-pulumi)
- [Additional complexity without the promised benefit: Terragrunt](#additional-complexity-without-the-promised-benefit-terragrunt)
- [Conclusion](#conclusion)

# Introduction

Public cloud computing has revolutionized the way organizations approach their IT infrastructure. 
Rather than investing in and maintaining their hardware and software, businesses can now access computing resources through the internet, on an as-needed basis. 
This shift towards public cloud adoption has created a demand for new management and deployment approaches, with Infrastructure as Code (IaC) emerging as a critical tool for cloud infrastructure management.

IaC is the process of defining and provisioning computing infrastructure through code, rather than through manual processes. 
This approach allows for consistent, repeatable, and scalable infrastructure deployment, while also providing the ability to automate management tasks and enforce governance policies. 
With IaC, developers can write code that defines the desired state of infrastructure, which can then be deployed and improved over time. 

As organizations increase their footprint in the cloud, the need for IaC becomes increasingly important. 
Without proper automation and governance, manual infrastructure management becomes time-consuming, error-prone, and difficult to scale. 
Developers can easily collaborate and share their infrastructure code, while also ensuring that resources are deployed consistently and securely. 
By automating infrastructure provisioning and deployment, organizations can more easily scale resources up or down as needed, optimize resource usage, and rapidly respond to changing business needs.

In this blog post, we'll discuss 3 different IaC tools: Terraform, Pulumi and Terragrunt. 
We'll discuss two real-world cases where Pulumi and Terragrunt were replaced by Terraform. 
We'll explain why they weren't the correct fit and what lessons we learned from using them.
If you're already familiar with the 3 tools, feel free to skip the next 3 sections and dive straight into the [use cases](#use-cases)

## What is Terraform?

Terraform is an open-source IaC tool, created by Hashicorp, that enables developers to provision and manage cloud infrastructure across various cloud providers. 
It uses a declarative language to define infrastructure resources, allowing developers to specify the desired state of resources such as virtual machines, load balancers, databases, and more. 
The tool then manages the entire lifecycle of these resources, from initial provisioning to improvements over time to deletion.
Terraform's use of code to manage cloud infrastructure provides several benefits. 

First, it allows for consistent and repeatable infrastructure deployment, eliminating manual errors.
Second, it enables collaboration and version control, as infrastructure code can be shared among teams and tracked through Git repositories. 
This allows teams to execute infrastructure changes using the same processes as code changes in their application.
Deploying a single instance of an application or 100s becomes as simple as a code change.
Terraform's versatility and portability make it an attractive choice for managing multi-cloud environments.
Its state management features ensure that infrastructure changes are auditable and transparent. 
With Terraform, organizations can achieve greater efficiency, scalability, and agility in their public cloud infrastructure management.

## What is Pulumi? 

Pulumi is another IaC platform that allows developers to define and manage cloud infrastructure using familiar programming languages such as Python, JavaScript, and Go.
Pulumi takes a different approach compared to Terraform, which uses a declarative language to define infrastructure resources.

With Pulumi, developers can create infrastructure resources using a variety of programming languages, leveraging the full power of those languages to manage infrastructure. 
This approach provides more flexibility and control compared to Terraform, as developers can use programming language constructs such as loops, conditionals, and functions to create more dynamic and complex infrastructure resources.
While Pulumi provides more flexibility and control compared to Terraform, it may also require more programming knowledge and experience to use effectively.
Terraform, on the other hand, provides a simpler and more standardized approach to infrastructure management, which can be easier for beginners to learn.

In essence, Pulumi and Terraform aren't that different, they just use a different `interface` to determine what the desired state of the infrastructure is. 
Pulumi even uses the Terraform libraries in the backend to create the desired state model.

## What is Terragrunt?

Terragrunt is a thin wrapper around Terraform that provides extra functionality and simplifies the management of multiple Terraform modules.
It is essentially a tool for managing Terraform code and configurations, and it uses a similar syntax to Terraform.

One of the main benefits of Terragrunt is that it provides a more modular approach to infrastructure management compared to Terraform. 
With Terragrunt, developers can define common configurations and modules that can be reused across multiple Terraform projects.
This makes it easier to maintain consistent infrastructure across an organization and reduces duplication of effort.

Another key benefit of Terragrunt is that it supports the automatic generation of Terraform configuration files, making it easier to manage and scale large infrastructure projects. 
It also includes a feature called "apply-all", which applies Terraform changes across all configured environments, simplifying the management of complex environments.
While Terragrunt provides several benefits, it does come with a learning curve, as it requires developers to learn a new syntax and understand its unique features.
Additionally, it adds another layer of complexity to infrastructure management, which may not be necessary for smaller projects.

# Use cases

## Developer adoption, excluding the batteries: Pulumi

### Background

This story starts with the IT operations team of a large corporation.
The team is responsible for managing all on-premise infrastructure and supporting teams in using public cloud providers such as Amazon Web Services (AWS) and Microsoft Azure.
For the scope of this story, the on-premise side of things isn't relevant. 

For the public cloud resources, the team supplied the development teams with accounts to manage and allowed them to either request infrastructure from the operations team or manage their infrastructure on their own. 
This meant that some development teams chose AWS CloudFormation to manage their infrastructure, some vendors supplied scripts (including Terraform) and some teams clicked the setup together in the AWS Console. 

The issue where temporary proofs-of-concept (PoCs) became `no such thing as temporary` production environments, was also a common occurrence.
Due to this approach, the governance and security footprint of applications depended highly on the AWS knowledge available in the teams.

Finally, just a side note, the CI/CD solution consisted of AWS CI/CD services: CodeBuild, CodePipeline, CodeCommit and CodeArtifact. 

### Managing expectations: Utopia

To improve governance and security of the landscape, standardized building blocks would be built.
These building blocks would adhere to the security and governance requirements by design. 
The idea was that project teams could use the building blocks directly in their project and could maintain them over time. 

The operation team evaluated multiple tools, including Terraform and Pulumi. 
As the project teams generally didn't know Terraform, but did know programming languages, Pulumi was chosen as a standard tool to build and distribute the building blocks. 

Pulumi supports a handful of languages at the time of writing: Go, Python, NodeJS (JavaScript/TypeScript), .NET and Java.
One of the assumptions made by the operations team was that the development teams would know at least one of these languages. 

Another advantage, especially compared to AWS CloudFormation or plain Terraform, is that programming languages generally have great support for testing, especially unit testing. 
This would allow the operations team to create components, validate them using traditional testing methods and supply components to teams with a high level of confidence that they would work as intended. 

Finally, the last relevant assumption for this story was the assumption that development teams would like to manage their infrastructure and would maintain the code needed to deploy, run and maintain that infrastructure. 

By now you might have figured out that these assumptions might be the cause of our `change of heart`

### Change of heart

The change of heart occurred after multiple difficulties and unexpected limitations.

The first red flag was discovered during the evaluation phase. 
While setting up Pulumi, most of the documentation assumes that the Pulumi Service product is used. 
This is most visible in the default behavior of the `pulumi login` command.
This command is needed to initialize the Pulumi context and its default behavior is to login into the Pulumi Service product.
It's possible to use AWS S3, Azure Storage Accounts or local files instead [as is documented here](https://www.pulumi.com/docs/intro/concepts/state/#using-a-self-managed-backend){:target="_blank" rel="noopener noreferrer"}. 
> Note that for AWS, profile support is implemented by adding it to the query parameters of the S3 and KMS connection strings. This has implications on the "shareability" of the configuration across developers. 

As there was support for the desired self-managed approach with some additional configuration, this wasn't considered an issue at the time.

During the implementation of the pipelines to deploy the Pulumi setup, another red flag occurred.
Reviewing infrastructure rollouts is a key feature of any IaC tool.
(Infrastructure) Developers want to be able to review the changes Pulumi wants to make to the infrastructure to match the new desired state. 
This feature has proven to be invaluable in the past to prevent costly or even unrecoverable mistakes in the code. 
Unfortunately, this is another area where Pulumi isn't strong at the time of writing. 
The support for reviewing the changes that Pulumi will make is limited to the CLI and saving a `change plan` that can be reviewed is supported, [but behind an experimental flag](https://www.pulumi.com/docs/intro/concepts/update-plans/){:target="_blank" rel="noopener noreferrer"}, which suggests it's not ready for production use. 
Currently, the change plan is also not in a review-friendly format.

Remember the assumption that development teams would know a programming language that was supported by Pulumi?
Good news, they do and Pulumi supports their language of choice.
The problem is that the most commonly used language is Java, the language that is currently only experimentally supported by Pulumi. 
This shouldn't be a problem as Pulumi supports cross-language component creation. 
This means that the operation teams can create Pulumi components in Python for example and allow development teams to use them in Java once that reaches prime-time status. 
Unfortunately, a theme starts to occur in the feature set of Pulumi. 
The cross-language component support is also a feature that's [still under heavy development](https://github.com/pulumi/pulumi/issues/6804){:target="_blank" rel="noopener noreferrer"} and cannot be used properly at the time of writing. 

Finally, for the sake of completeness, there were a few additional issues that are worth mentioning.

The assumption that (unit) tests could be easily written to validate the Pulumi code, doesn't hold true today. 
There is [support for testing](https://www.pulumi.com/docs/guides/testing/){:target="_blank" rel="noopener noreferrer"} in Pulumi, but due to the lack of `real` mocking capabilities (including any side-effects) in Pulumi, the tests feel like testing getters and setters instead of proper tests. 
A solution like the mocking server in the [Fabric8 Kubernetes client](https://github.com/fabric8io/kubernetes-client#mocking-kubernetes){:target="_blank" rel="noopener noreferrer"} would solve this issue.

Another side-effect of having a regular programming language is that the order of execution of statements is important. 
When the output of a previous statement is needed as input for the next, one cannot simply use the output of the first command to pipe it to the next. 
This is a limitation because Pulumi needs to build a desired state model in the backend before it can know the actual output for some statements. 
When the output of a statement is needed in the next statement, it needs to have a way to postpone the execution of that statement.
[Pulumi does supply helper functions](https://www.pulumi.com/docs/intro/concepts/inputs-outputs/#outputs-and-strings){:target="_blank" rel="noopener noreferrer"} to make this less of an issue, but it's something that needs to be considered when developing the code.

### Pulumi to Terraform conclusion

These previous issues combined caused the team to switch to Terraform as their preferred IaC tool.

Pulumi has a ton of potential and might become the next big multi-cloud IaC tool in the coming years. 
Unfortunately, too many parts don't feel ready enough to be used in a corporate context where infrastructure developers come at a premium.

Terraform has a proven track record. 
It supports shared modules in a variety of ways: S3, Git, and local files, ... 
It properly supports a split between the `plan` and `apply` phases, meaning that the change plan can be reviewed.
Terraform Cloud is a SaaS solution provided by Hashicorp, but it doesn't feel like a requirement to properly use Terraform. 

Terraform isn't perfect either though.
There is no real testing automated capability in Terraform, especially without deploying the infrastructure.
Developers need to learn yet another language (HCL) to use Terraform and it doesn't have the flexibility of a real programming language.

## Additional complexity without the promised benefit: Terragrunt

### Background

When starting a new infrastructure project, it's important to choose the right tools for the job. But what happens when you inherit an existing project with infrastructure as code (IaC) already in place, using a tool like Terragrunt that your team is not familiar with? 
This is the situation the team in this story found themselves in.

The project they were working on was based on an existing infrastructure stack that had been built using Terragrunt, a popular wrapper around Terraform. 
While Terragrunt can be a powerful tool for managing complex infrastructure stacks, the new team found that it added unnecessary complexity and overhead to their workflow. 
They struggled to read and understand the existing Terragrunt code, which had been split into multiple Terraform modules in different git repositories and brought back together using a Terragrunt configuration repository.

Adding to the complexity, the Terragrunt configuration was loading different versions of the Terraform modules in different environments, and the Terraform state was split into modules as well, with custom scripts to read and manipulate the state. 
All of these factors made it difficult for the new team to make progress on the project, and they began to question the decision to use Terragrunt in the first place.

### Deciding to Return to Terraform

After struggling to work with the complex and fragmented Terragrunt setup, the new infrastructure team wanted to find a simpler, more streamlined approach. 
They sought a setup with a low threshold, low maintenance, and easy to understand.

As they dug deeper into the Terragrunt configuration, they found that even the company's architects had some doubts about its usefulness. 
So they began discussing alternatives, and eventually, someone asked the question: "Why are we using Terragrunt?"

The answer was surprising: the team wasn't even using any of Terragrunt's real advantages and was instead dealing with unnecessary overhead. 
The configuration of all external modules was just a single key-value file, with no clear indication of which value was passed to which module.

It became clear that continuing to use Terragrunt was simply pointless, and only adding to the team's frustration. 
They, together with the company's architects, decided to switch back to using Terraform directly, to simplify their workflow and make progress on the project.

### Returning to terraform without breaking the existing infrastructure

After deciding to switch back to using Terraform directly, the team realized that a complete overhaul of their infrastructure setup would be necessary. 
Rather than throwing out all of the existing Terragrunt code, however, they decided to take a hybrid approach.

The first step was to set up a new mono git repository that would contain all of the different Terraform modules that had previously been spread across multiple repositories. 
They also created a new Terraform root configuration setup, which would enable them to manage the entire infrastructure as a single entity.

Next, the team began moving these Terraform modules into their new mono repo. 
As they did so, they took the opportunity to clean them up, removing any unused input variables or features. 
They also improved the security setup, as the project required the infrastructure to be publicly accessible instead of hidden behind an on-premises network.

By consolidating all of the Terraform modules in one place, the team made it easier for existing teams to import them into their Terragrunt setup. 
The hybrid approach proved to be a successful strategy, enabling the team to manage their infrastructure more effectively and efficiently, but also not breaking what was already in place.

### Some other (un)expected advantages

Moving to a mono repo from the existing Terragrunt setup not only simplified the project's infrastructure but also brought several expected and unexpected advantages.

One of the most significant issues with the Terragrunt setup was its complexity and slow performance on the CI/CD pipelines, which checked out multiple git repositories. 
However, by consolidating the Terraform modules into a single repository, the team eliminated the need to check out different repositories during CI/CD, significantly speeding up the pipeline.

The development process also benefited from the switch to a mono repo. 
A simple change to the infrastructure no longer required updates to multiple git repositories, making the process more efficient and reducing the chance of errors.

In addition to these expected advantages, the team discovered other unexpected benefits. 
For example, setting up a new environment only requires a simple configuration file with around ten configuration values. 
This means that setting up a new environment is much quicker and more straightforward than before.

Overall, the decision to move from Terragrunt to a hybrid approach with a mono repo proved to be a wise one. 
The team enjoyed a simpler, more efficient infrastructure setup, improved performance on CI/CD pipelines, and unexpected benefits such as a streamlined environment setup.

### Conclusion

The decision to move away from Terragrunt and towards a hybrid approach of using Terraform and a mono repo had several (un)expected benefits for the team. Not only did it simplify the setup and reduce maintenance overhead, but it also significantly sped up the development process and CI/CD pipelines.

The team was able to clean up and improve the security of the existing Terraform modules, and consolidate them into a single repository, making it much easier to manage and make changes. The move to a simpler configuration file for new environments also made it easier to spin up new instances of the infrastructure.

While it can be tempting to stick with existing tools and setups, sometimes a reevaluation of the current setup can lead to significant improvements in productivity and efficiency.

# Conclusion

The main takeaways of this blog post are twofold: 
- use the right tool for the job, technically and,
- even more importantly, use a tool that the users can understand and/or have knowledge on

Often tools are selected based on a first impression and a very small PoC, as in the first story. 
These PoCs are often a good indicator of the feasibility of a tool, but they don't show real use of course.
Therefore we recommend including the maturity of a tool and the general availability of knowledge about the tool in the market. 
Look up how many TerraGrunt or Pulumi developers there are near you and compare that to the number of Terraform developers you can find. 

Next to selecting the right tool for the right reasons, your existing team also needs to be able to work with the tool.
Selecting a tool therefore as much of a technical as a people decision. 

Finally, selecting a well-established tool is always a safe bet. 
We'd like to highlight that you can and should select more innovative and less established tools as well but don't use them for your core business or as something that will support (or fail to) your entire application landscape.
Pulumi and TerraGrunt might pop up on our horizon in the future again, but for now, we'll stick with good old ([28 July 2014](https://en.wikipedia.org/wiki/Terraform_(software){:target="_blank" rel="noopener noreferrer"})) Terraform. 

If you want help with your Infrastructure As Code, cloud adoption in your company or want to provide feedback, feel free to contact [Yannick](https://be.linkedin.com/in/yannick-bontemps-885379b0){:target="_blank" rel="noopener noreferrer"} or [Pieter](https://be.linkedin.com/in/pieter-vincken-a94b5153){:target="_blank" rel="noopener noreferrer"} on LinkedIn!