# uSIEM a framework to build SIEMs

First of all, the project is hosted here: https://github.com/u-siem

### A bit of context
After a long time thinking about what is wrong with the current SIEM ecosystem, I have decided to build my own. Well, not exactly, uSIEM (micro SIEM) is a small but powerful framework for building a custom SIEM with performance and robustness at its core.

With my experience as DevOps and my passion for automating and incorporating testing into everything, I have designed a simple system that can scale (probably) into a monster. You can use uSIEM as a full-featured SIEM, as an ingestion tool to replace Logstash (which requires an outrageous amount of resources), or use its open libraries to analyze logs in third-party tools.

You can select any type of input (syslog, elastic like API, database connectors), use data sets (like QRadar reference sets but with the things that make uSIEM great), or select the type of output (where store the logs) you want: ElasticSearch, MySQL, Postgres ... or None if you just want to apply rules. Think of uSIEM as a small component or LEGO pieces that can build complex or simple things sturdily.

To achive this, the first step is the Parsing and Normalization of sources. To see why uSIEM is so special (or at least for me) we can look at some unit tests (https://github.com/u-siem/usiem-squid):

```rust
#[test]
fn test_log_from_syslog() {
    let log = "<1>1 2020-09-25T16:23:25+02:00 OPNsense.localdomain (squid-1)[91300]: 1601051005.952  18459 192.168.4.100 TCP_TUNNEL/200 7323 CONNECT ap.lijit.com:443 - HIER_DIRECT/72.251.249.9 -";
    let log = SiemLog::new(log.to_string(), 0, SiemIp::V4(0));
    match super::parse_log(log) {
        Ok(log) => {
            assert_eq!(log.field(field_dictionary::SOURCE_IP), Some(&SiemField::IP(SiemIp::from_ip_str("192.168.4.100").expect("Must work"))));
            assert_eq!(log.field(field_dictionary::DESTINATION_IP), Some(&SiemField::IP(SiemIp::from_ip_str("72.251.249.9").expect("Must work"))));
            assert_eq!(log.field(field_dictionary::EVENT_OUTCOME), Some(&SiemField::from_str("ALLOW")));
            assert_eq!(log.field(field_dictionary::HTTP_RESPONSE_STATUS_CODE), Some(&SiemField::U64(200)));
            assert_eq!(log.field(field_dictionary::URL_DOMAIN), Some(&SiemField::from_str("ap.lijit.com")));
            assert_eq!(log.field(field_dictionary::DESTINATION_PORT), Some(&SiemField::U64(443)));
            assert_eq!(log.field(field_dictionary::DESTINATION_BYTES), Some(&SiemField::U64(7323)));
            assert_eq!(chrono::NaiveDateTime::from_timestamp(log.event_created(),0).to_string(),"2020-09-25 16:23:25");
        },
        Err(_) => {
            panic!("Cannot parse log")
        }
    }
}
```
All the code is done in Rust, that allows me to build robust code, and for me its seems like Rust is a heaven made language for parsing logs, because you could never forget to check if a variable (or Log field) is Null.

All changes done to the source code needs to pass a a CI/CD pipenline that simply does a `cargo test`: 
![Rust](https://github.com/u-siem/usiem-squid/workflows/Rust/badge.svg?branch=main)

In the future, because many tools are open source like Squid, Snort, Apache, Nginx ... we could create docker images and test the parsers with real logs. I haven't created the integration tests yet because I don't have enough time, but the Docker images are already created. 

And also, we could test rules in real environments using Windows VMs with Github Actions or in private labs.

### Benefits

One of the benefits of parsing records using pure code and no regular expressions is the number of records that can be processed with one thread. I have a repository to test the performance of each parsing library https://github.com/u-siem/parser-benchmarks take a look at the numbers: 

| Library        | Source           | Modules  | EPS             |
| -------------- |:----------------:|:--------|:----------------:|
|[usiem-apache-httpd ](https://github.com/u-siem/usiem-apache-httpd)| Apache2 | IPS, WebServer | ¿?¿?¿? |
|[usiem-paloalto ](https://github.com/u-siem/usiem-paloalto)| PaloAlto FW | Firewall, IPS, WebProxy, Auth, Endpoint | 260.596 |
|[usiem-sonicwall ](https://github.com/u-siem/usiem-sonicwall)| SonicWall FW | Firewall, IPS, WebProxy, Auth, Endpoint | 180.727 |
|[usiem-opnsense ](https://github.com/u-siem/usiem-opnsense)| OpnSense FW | Firewall | 475.239 |
|[usiem-windns ](https://github.com/u-siem/usiem-windns)| Microsoft DNS Server | DNS | 680.920 |
|[usiem-squid ](https://github.com/u-siem/usiem-squid)| Squid Proxy | WebProxy, IPS | ¿?¿?¿? |

And this is for single thread, of course, this is just parsing performance, but if you are familiar with SIEMs you know they can barely get to 20K events per second withouth investing lots of money.

Another cool thing is the standarization of event types, https://github.com/u-siem/u-siem-core/blob/main/src/events/mod.rs

* Firewall: has information about IP connections.
* Intrusion: IDS / IPS related logs like Suricata, Snort, OSSEC, Wazuh, NGFW ...
* Assessment: The result of vulnerability scanners or policy makers.
* WebProxy: browser proxy information.
* WebServer: web application servers, adaptive content distribution or LoadBalancers for HTTP traffic.
Sandbox: Like an antivirus, a Sandbox retrieves information about a file that is malicious or not. It can be used to extract file names, hashes, or other relevant information to update a dataset of known hashes and trigger queries.
* Antivirus: antivirus related events: virus detected / blocked ...
* DLP: data loss prevention
* Partitioning: some devices like email gateways generate a large number of records when an email arrives: header processing, AV scanning, attachment information ... In those cases, each record is associated with an action by an ID of tracking or a transaction ID.
* EDR: Endpoint detection and response devices, also EPP.
* Mail: Mail events. Eg: Microsoft Exchange, IronPort, Office 365 ...
* DNS: used to detect queries to malicious sites.
* DHCP: records that associate an IP with a MAC address.
* Auth: records related to authentication. Logins to devices or services.
* Endpoint: local events related to servers or workstations: operating system events, cleaned log files, user / group modification, firewall rules ... 

Each of these types will go into a processing pipeline that will enhance them: assign the hostname if the IP is local and in the dataset containing the IP <-> HOSTNAME pair, save the information using DHCP records in a data set for later use, check the malice of an IP address ... 

Also the normalization allows to search for the same things in different device types, a good example is the WebProxy event:

```rust
#[derive(Serialize, Debug, PartialEq, Clone)]
pub enum WebProxyRuleCategory {
    Abortion,
    MatureContent,
    Alcohol,
    AlternativeSpirituality,
    ArtCulture,
    Auctions,
    AudioVideoClips,
    Trading,
    Economy,
    Charitable,
    OnlineChat,
    ChildPornography,
    CloudInfrastructure,
    CompromisedSites,
    InformationSecurity,
    ContentDeliveryNetworks,
    ControlledSubstances,
    Cryptocurrency,
    DynamicDNSHost,
    ECardInvitations,
    Education,
    Email,
    EmailMarketing,
    Entertainment,
    FileStorage,
    Finance,
    ForKids,
    Gambling,
    Games,
    Gore,
    Government,
    Hacking,
    Health,
    HumorJokes,
    Informational,
    InternetConnectedDevices,
    InternetTelephony,
    IntimateApparel,
    JobSearch,
    MaliciousOutboundDataBotnets,
    MaliciousSources,
    Marijuana,
    MediaSharing,
    Military,
    PotentiallyAdult,
    News,
    Forums,
    Nudity,
    BusinessApplications,
    OnlineMeetings,
    P2P,
    PersonalSites,
    PersonalsDating,
    Phishing,
    CopyrightConcerns,
    Placeholders,
    PoliticalAdvocacy,
    Pornography,
    PotentiallyUnwantedSoftware,
    ProxyAvoidance,
    RadioAudioStreams,
    RealEstate,
    Reference,
    Religion,
    RemoteAccess,
    Restaurants,
    QuestionableLegality,
    SearchEngines,
    SexEducation,
    SexualExpression,
    Shopping,
    SocialNetworking,
    DailyLiving,
    SoftwareDownloads,
    Spam,
    Sports,
    Suspicious,
    Technology,
    Tobacco,
    Translation,
    Travel,
    VideoStreams,
    Uncategorized,
    URLShorteners,
    Vehicles,
    Violence,
    Weapons,
    WebAds,
    WebHosting,
    WebInfrastructure,
    Others(String),
}
```
All web proxies that support URL categorization must follow these categories. With this we could search the same URL Categories in different Proxies, this is really usefull in multi-client environments.


### Next steps

I am planning to continue developing uSIEM step by step, but as now it cannot be used in production before v1.0.0 arrives.
For now the roadmap has SIGMA rules, ElasticSearch input / output support, the Kernel component that ties the rest together, and an extensible user interface. 
If you are interested in the project and want to share ideas, feel free to contact me or make a pull request to the project.
