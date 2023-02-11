---
name: Multiple BGP Labs using Infrastructure as Code
tools: [Development]
image: /media/2022-02-11/bgp-lab.jpg
description: Explore my BGP Lab Project!
---

<img src="/media/2022-02-11/bgp-lab.jpg">

Photo by <a href="https://unsplash.com/@marekpiwnicki?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Marek Piwnicki</a> on <a href="https://unsplash.com/photos/QajN7imAkyI?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  
  
There is a fair amount of documentation regarding these labs in my [Github](https://github.com/RFC2516/clab-bgp). A quick summary on what's included,

This is a repository that I am creating to teach myself unit testing, portable infrastructure as code, configuration management and labbing content for the Cisco ENCOR Exam. I have provided all necessary files to complete the lab except for the Cisco CSR 1000v image. You must obtain your own Cisco CSR 1000v image legally via Cisco. After obtaining your image you can use the vrnetlab project to containerize it for Containerlab.

[ContainerLab](https://containerlab.dev/) represents my Infrastructure as Code (IaC) orchestrator tool. You can get started with their install guide [here](https://containerlab.dev/install/). I am using Windows Subsystem for Linux as my local host. If you skim over their installation guide there is a warning provided in the documentation appears to be referring to WSL version 1. If you too choose to use the Windows Subsystem for Linux to run this project then ensure you are using [WSL version 2](https://learn.microsoft.com/en-us/windows/wsl/install#check-which-version-of-wsl-you-are-running) like I am, I have not faced an issues using WSL version 2.

For an example of the project in action check our this asciinema replay:

[![asciicast](https://asciinema.org/a/553697.svg)](https://asciinema.org/a/553697?autoplay=1)