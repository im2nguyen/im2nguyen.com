---
title: "What Does an Education Engineer Actually Do"
date: 2021-12-10T11:31:36-08:00
draft: true
---

Over Thanksgiving, my family had questions about what I do at work. In previous 
years, they were satisfied with "software engineer" — I made websites and apps. 
They weren’t sure what an education engineer actually does.

![Do you code or teach? Yes. Comic](/img/education-engineer-comic.png)

At HashiCorp, education engineers design scenarios that emulate problems DevOps 
engineers may run into and guide them through a solution. As part of the 
Terraform Education team, these scenarios may come in the form of common use 
cases, like how to [spin up a Kubernetes cluster](https://learn.hashicorp.com/tutorials/terraform/eks) 
in your favorite cloud, or highlighting new features practitioners may find 
useful, like [curating public providers and modules within Terraform Cloud](https://learn.hashicorp.com/tutorials/terraform/private-registry-add?in=terraform/modules).
Since education engineers need to generate real world examples and their 
solutions, they’re expected to have real experience and technical knowledge. If 
you have prior experiences in advocating for and implementing [blue-green deployments](https://learn.hashicorp.com/tutorials/terraform/blue-green-canary-tests-deployments), 
you’re intimately aware of both the benefits and hurdles. 

In the ["grand unifying theory of documentation"](https://documentation.divio.com),
 education engineers help practitioners develop a mental model by creating 
 content that’s [learning](https://documentation.divio.com/tutorials/) and 
 [problem-orientated](https://documentation.divio.com/how-to-guides/). The 
 Terraform [Get Started](https://learn.hashicorp.com/collections/terraform/aws-get-started) 
 collections show users Terraform’s immediate value by having users provision, 
 modify and destroy a compute instance through code. Once users realize how to 
 use the product, they may navigate to content that covers the [specific use case](https://learn.hashicorp.com/collections/terraform/use-case) 
 they’re trying to address. By consuming content that’s learning and problem-orientated, 
 practitioners are better able to dive into and navigate the documentation to 
 solve their unique problems.

![Grand unifying theory of documentation quadrant broken down into tutorials, how-to guides, explanation and reference](/img/unified-theory-documentation.png)

While all of the content lives in our education platform, [Learn](https://learn.hashicorp.com), practitioners can engage with it in different ways:

  1. **Tutorials.** The majority of the content we create are tutorials, which 
     guide practitioners through a scenario. Most tutorials include a set of 
     commands and explanations that practitioners can copy and paste into their 
     terminal. We design the scenarios so that on completion, practitioners can 
     apply what they learned to a problem they’re facing. Tutorials serve as 
     our foundation — all additional content (e.g. interactive labs, workshops) 
     must stem from a tutorial. This gives practitioners an opportunity to 
     review and re-run sections they’re stuck with while running through the 
     lab or workshops.
     
     We select new topics for tutorials from a combination of metrics that help 
     us assess whether a tutorial could be popular. This serves as an imperfect 
     proxy that measures whether users find the scenario both relevant and 
     engaging. For some tutorials, we work closely with engineers, product 
     managers, and other stakeholders to serve as the voice of the practitioner. 
     Finally, the team rigorously tests and reviews all tutorials — an average 
     tutorial review cycle has 100+ suggestions)!
     
     ![Example Learn tutorial page with command snippet](/img/learn-tutorial.png)
  
  2. **Interactive Labs.** For some tutorials, we create interactive labs with 
     [Instruqt](https://play.instruqt.com/hashicorp-learn) and then embed these 
     labs into our tutorials. This enables practitioners to experiment with 
     HashiCorp product features without having to install any dependencies. 

     ![Example Learn tutorial page with embedded lab](/img/embedded-instruqt-learn.png)

  3. **Workshops.** During HashiConf, HashiCorp main conferences, the education 
     team hosts hands-on workshops. In these workshops, we guide practitioners 
     through interactive labs while answering questions they may have about the 
     feature/product or the scenario itself. 

During HashiConf Global 2021, we guided attendees through building a golden 
image pipeline with HCP Packer, a new HashiCorp product offering that was 
announced the day prior. By offering a workshop shortly after the product 
launch, we were able to effectively get users to try the product. 

In addition to the content on Learn, as an education engineer, we’re encouraged 
to work on problems that the community struggles with. While working with 
Terraform, I noticed that it was difficult to quickly get a graphical overview 
of changes. This is especially important if you’re managing a large number of 
resources, cross multiple regions and providers. When I proposed a visualizer 
solution that could address this concern, my manager, Judith, encouraged me to 
work on it even though it wouldn’t directly increase the number of visitors to 
Learn. When I shipped the [first version](https://github.com/im2nguyen/rover), 
Kerim, a developer advocate, encouraged me to speak about my experiences 
building it during [HashiTalks: Build](https://www.youtube.com/watch?v=zIwZ6XEeCAo). 
Rover wouldn’t exist without their support and encouragement.

If this experience resonates with you and you enjoy working with amazing people, 
the [HashiCorp Education team is hiring](https://www.hashicorp.com/jobs/developer-relations)! 
Feel free to DM me [@im2nguyen](https://twitter.com/im2nguyen) if you want to 
learn more.

## Additional Resources

Geoffrey, Director of Product Education Engineering at HashiCorp, wrote an 
insightful article titled [What Does an Education Engineer Do?](https://topfunky.com/2020/education-engineer-career). In it, he describes the skills and expectations for 
each level of education engineers.

