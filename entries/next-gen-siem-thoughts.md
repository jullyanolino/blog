# uSIEM NextGen SIEM or alredy dead?
Today reading some thoughts about SIEMs from Anton Chuvakin inspired me to continue building uSIEM. 
For those of you who do not know, it's a event driven framework based on modularity, performance and sturdiness above anything else that allows you to build a SIEM.
Sounds good? It is. Think about getting up a full SIEM with rules and parsers. It's really a long process full of pain. Thats where uSIEM shines, you only need a `git clone custom_usiem_repo` and a `cargo build --release` to have a ready to deploy SIEM. Cool right?

### Modularity
The SIEM is divided in components and a kernel. A component, like a linux process, uses pipes, receive data from an input and sends data to an output. This data can be commands, logs, datasets or querys. The kernel is responsible of interconnecting the inputs and outputs of one component to the next one. Also it delivers the messages like commands and queries from one component to another.

The first component in the pipeline is the **receiver** of logs like a syslog server, the next one is the **parser**. Obviusly uSIEM will provide a default parser component, but it can be replaced with a custom one. 

After the parser comes the **enchancer**. It's behaviour must be fixed, it operates the datasets adding or removing elements from them, duplicates fields or adds missing ones to logs, like `url.path`,  `url.domain` or  `url.query` extracted from `url.full`. For WebServers it checks and converts if necessary the event type from WebServer to Incident if it detects a SQLi or XSS in the url fields. For DHCP events it adds to a dataset the association of MAC/IP or removes it. Also for Database types like MySQL that logs the entire Query, it can check if it contains a SQLi to trigger alerts.
Finally the log is "duplicated" and sent to 3 components (this is optional):
* One is the **indexer** that can send logs to a database like elasticsearch, mysql, sqlite... The indexer needs a data Schema to know what to filter out of the database. This schema must be built joining the schemas of each parser (The kernel will do this for you by default).
* Another one is the **rule engine** that triggers alerts based on rules. The default one will be based on SIGMA rules.
* Finally the **behavior engine**. The idea of this component is to send logs to external systems to extract patterns and trigger rules. It can be anything you want: machine learning, graphics ... This is the only component that uSIEM will not provide.

There are other ideas on the table, like lightweight components that don't need to run within a thread because they only do simple things like just provide an HTML or an action like Block an IP on a firewall (Yes! SOAR capabilities). 

### Performance and sturdiness

uSIEM runs away from Regex (but can be used when you really need to) so, it uses loops to parser and extract fields from logs. I have read in the [Splunk blog](https://www.splunk.com/en_us/blog/it/how-splunk-is-parsing-machine-logs-with-machine-learning-on-nvidia-s-triton-and-morpheus.html) about parsing logs with machine learning because REGEX can fail to parse certain logs when a change was made. Completly true and more in the case of Cisco ASA logs. I remember that one particular log in Cisco ASA changed and added a whitespace where there was alredy one, so because the regex was built with "\s" instead of "\s+" the parser failed to extract fields. Checking why the parser was falling was really funny (of course not).

The aproach of uSIEM is different from Splunk, the parsers will return an Error (different from throwing an Error) of ParserError so the parser component will know that the log has changed and will alert the admin of the change in format. 
If you want real numbers check [Parser Benchmarks](https://github.com/u-siem/parser-benchmarks) to get an idea of the events/second that can be delivered with only one thread.

### Testing and automation

Integrating security and testing in the same phrase was never that easy to achieve. With uSIEM because you could execute each component withouth the rest you could test the parser and the rule engine at once.
Imagine having in a separate repository the rules and in another the SIEM and joining them with a CI/CD tool. We could have integration tests (Check [uSIEM Squid](https://github.com/u-siem/usiem-squid/blob/main/tests/integration.rs)) or rule tests that with a bunch of logs inside a JSON could check that an alert is triggered. 
If you are purple I'm sure you will be asking... Can you test it in a real Windows environment? YES!! Thats the power of CI/CD + uSIEM, deploying a new version of the SIEM and starting a script that emulates an attack to check if certain alerts are triggered.


#### Some SIEM literature
https://medium.com/anton-on-security/security-correlation-then-and-now-a-sad-truth-about-siem-fc5a1afb1001
https://medium.com/anton-on-security/today-you-really-want-a-saas-siem-1b980b627ba9
https://medium.com/anton-on-security/modern-siem-mysteries-80fcd699da68
https://medium.com/technology-hits/the-fundamental-and-future-of-siem-52cfc1dd4e7a
https://www.splunk.com/en_us/blog/it/how-splunk-is-parsing-machine-logs-with-machine-learning-on-nvidia-s-triton-and-morpheus.html