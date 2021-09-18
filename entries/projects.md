## Summary of my projects

#### [Cancamusa](https://github.com/SecSamDev/cancamusa)

Cancamusa lets you design a full Windows environments with workstations and servers. Once it meets our desired state, Cancamusa allows you to first build all the files and then deploy it in a Proxmox server (the tool must be installed in the server).

It's still under development, but for the moment, it has helped me a lot reducing the time needed to build a custom Windows laboratory (only 20 min needed).

You can download and deploy some examples here: https://github.com/SecSamDev/cancamusa-labs


#### [uSIEM](https://github.com/u-siem/)

uSIEM is a still-in-development framework that allows you to develop a custom SIEM. Think of it as Spring Boot but for SIEM.

It has modern things like dynamic data sets (in QRadar slang reference sets), it is meant to be as simple as possible and extremely flexible to be able to adjust to different types of customers and needs.

The core idea is that you will be able to clone a repository with all the SIEM code and be able to compile and run it. It will have all rules, parsers and things it needs to run.