---
layout: page
title: About
permalink: /about/
---

<style>
  main ul, main ol {
    margin-left: 30px;
    padding-left: 35px;
    line-height: 1.5;
  }

  main li {
    margin-bottom: 10px;
  }

  div.sect {
    overflow: auto;
    margin-top: 30px;
  }

  div.sect > img.portait {
    width: 25%;
    padding-right: 15px;
    padding-bottom: 10px;
  }

  div.sect > img.baobab {
    max-width: min(100%, 500px);
    padding-left: 15px;
    padding-bottom: 10px;
  }

  hr.dashed {
    margin: 40px 0px;
    border: 1px dashed #ddd;
  }
</style>

### Looking for help with a project that uses LLVM?

<div class="sect">
  <img src="https://weliveindetail.github.io/blog-sandbox/res/2023-profile.jpg"
       class="baobab" align="left">
  <p>
    <b>As an independent contributor</b> I am actively <a href="https://github.com/llvm/llvm-project/commits?author=weliveindetail">working on LLVM</a>.
  </p>
  <p>
    <b>As a freelance developer</b> I help small and mid-size companies to get up to speed with LLVM. <a href="https://www.openstreetmap.org/search?query=berlin%20runge%20str.%2020#map=16/52.5127/13.4201">Here in Berlin</a> and remote.
  </p>
  <p>
    <b>I have a track record of remote work</b> with various companies since 2016 and offer flexible conditions to match your demands. Feel free to reach out for questions <a href="click:the.address.will.be.decrypted.by.javascript" onclick='openMailer(this);'>via email</a> or <a href="https://calendly.com/stefan-graenitz/30min" target="_blank">schedule a video call</a> right away.
  </p>
</div>

<hr class="dashed">

#### LLVM upstream contributions

<div class="sect">
  <p>
    <a href="https://weliveindetail.github.io/blog/res/2023-baobab-me-llvm.html" target="_blank">Interactive visualization</a> of my <a href="https://github.com/llvm/llvm-project/commits?author=weliveindetail">LLVM upstream contributions</a> rendered with <a href="https://github.com/weliveindetail/git-baobab" target="_blank">git-baobab</a>. Leaf nodes link to list of individual commits on GitHub:
  </p>
  <a href="https://weliveindetail.github.io/blog/res/2023-baobab-me-llvm.html" target="_blank">
    <img alt="2023 Baobab-Me LLVM" class="baobab"
         src="https://weliveindetail.github.io/blog-sandbox/res/2023-baobab-me-llvm.jpg" align="right">
  </a>
</div>

<hr class="dashed">

#### Setup

* Get your project started right now and build a solid foundation
* Set up build and test infrastructure that fits with upstream LLVM

#### Evaluations

* Evaluate feasibility and estimate efforts for your planning
* Solve tricky questions early on to limit risks

#### Porting

* Port your project to new platforms and architectures
* I have extensive experience with [Windows ports](https://www.reddit.com/r/cpp/comments/4l2mdd/juce_projucer_live_c_ide_has_been_coming_soon_for/d84gp1t/)
* IoT embedded and bare-metal projects are ramping up

#### Mentoring

* Train internal devs by working on ready-made tasks
* Have an expert at hand for for both mentoring and emergencies

#### Bugfixing

* Investigate problems with public toolchain releases
* Fix them upstream to limit future maintenance effort

#### Feature Development

* Prototype your new feature in a downstream fork
* Upstream it into public toolchain releases

<hr class="dashed">

#### Build a solid foundation for your project

* Evaluate feasibility and estimate efforts early on.
* Set up build infrastructure for using LLVM, Clang or any other subject.
* Bring up a test suite to use the [LLVM integrated tester](https://llvm.org/docs/CommandGuide/lit.html) for your own project.
* Port your project to new platforms -- [why not Windows](https://www.reddit.com/r/cpp/comments/4l2mdd/juce_projucer_live_c_ide_has_been_coming_soon_for/d84gp1t/)?

#### Manage your own changes on LLVM

* Guidance on maintaining a fork downstream of LLVM.
* Get in touch with the community and communicate your issues.
* Represent your point of view and argue for your interests in the community.

#### Upstream your changes to LLVM

* Advice on [upstream guidelines](https://llvm.org/docs/SupportPolicy.html), risks and chances.
* Bring up solid tests for your use cases.
* Review your changes in [Phabricator](https://reviews.llvm.org/).
* Deal with [build bots](http://lab.llvm.org:8011/#/console) and test failures.

#### Workshops and trainings on-site or online

* LLVM upstream development best practices
* C++, library-based design and error handling in LLVM

<hr class="dashed">

Compiler development is the royal discipline of computing. It always demanded the highest skills from its makers and always fascinated me the most as a developer. Compilers are well understood in academics since long, but mainstream implementations were often lacking behind today's standards -- being quirky, proprietary and hard to master.

With the advent of [LLVM](https://stackoverflow.com/questions/2354725/what-exactly-is-llvm){:target="_blank"} a lot of this changed and today we have state of the art modular compiler technology right at our fingertips -- with a [clean three-phase architecture](https://www.aosabook.org/en/llvm.html){:target="_blank"} [open-source](https://github.com/llvm/llvm-project/){:target="_blank"} and reusable.

In my spare time I am hacking on compilers and tools and I use this blog to share my experiences. If that meets your own interests please join our [LLVM Social Berlin](https://www.meetup.com/de-DE/LLVM-Social-Berlin/){:target="_blank"}, <a href="click:the.address.will.be.decrypted.by.javascript" onclick='openMailer(this);'>write an email</a> or catch me on <a href="https://twitter.com/weliveindetail" target="_blank">Twitter</a>.
