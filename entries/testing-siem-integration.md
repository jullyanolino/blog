# When CI/CD meets Security

Today was a day worth celebrating. A culmination of all the effort invested in uSIEM. 

Today I was able to test a component using real logs in a CI/CD pipeline.

Before we begin check [uSIEM Squid](https://github.com/u-siem/usiem-squid)

Extracting the logs from the container was simpler than I initially thought: setting a Lighttp that serve me the logs proxied by the Squid: `curl -x http://127.0.0.1:3128  -L http://127.0.0.1:80/deny.log`

From submitting code to github to compiling and running tests inside a virtual machine in Azure with a Squid docker. 
```
[git push] -> [github action triggered] -> [launch VM] -> [run containers] -> [cargo build] -> [cargo test]
``` 

And the test code is easy to follow:
```rust
#[test]
fn test_squid_integration() {
    // Use CI_CD var to enable the integration tests and to never fail when testing in a local environment
    let out_dir = env::var("CI_CD").unwrap_or(String::from(""));
    if out_dir == "" {
        return;
    }
    // Proxy the http request using Squid (port 3128)
    let client = reqwest::blocking::Client::builder()
        .proxy(reqwest::Proxy::http("http://127.0.0.1:3128").unwrap())
        .build()
        .unwrap();
    // SquidGuard is listening if the log is created
    let res = client.get("http://127.0.0.1:80/squidGuard.log").send().unwrap();

    if !res.status().is_success() {
        panic!("SquidGuard must be active");
    }

    // Request a URL marked as Hacking in the domainlist
    let hack_url = "http://hackpage.com/random-stuff/and-random.html?param_1=value_1&param_2=value_2";
    get_url(hack_url, &client);
    ...
    ...
    ...
    // Get the SquidGuard Deny LOG and check that the parser is working
    let res = client.get("http://127.0.0.1:80/deny.log").send().unwrap();
    if !res.status().is_success() {
        panic!("The URL deny.log MUST not be blocked. Error in configuration");
    }
    let deny_text = res.text().unwrap();
    let split = deny_text.split("\n");
    let deny_text: Vec<&str> = split.collect();

    let deny_hack = deny_text.get(0).unwrap();
    test_denied_hack(deny_hack);
```

The *test_denied_hack* simply checks that the values extracted using the parser were the ones we expect:
```rust
fn test_denied_hack(denied_text : &str) {
    //Create a log object received from the IP=0 and in the time=0
    let log = SiemLog::new(denied_text.to_string(), 0, SiemIp::V4(0));
    match squidguard::parse_log(log) {
        Ok(log) => {
            //The was not allowed, so the destination_ip = 0
            assert_eq!(log.field(field_dictionary::DESTINATION_IP), Some(&SiemField::IP(SiemIp::from_ip_str("0.0.0.0").expect("Must work"))));
            // The request was blocked
            assert_eq!(log.field(field_dictionary::EVENT_OUTCOME), Some(&SiemField::from_str("BLOCK")));
            // The event_code=503=Unavailable=Blocked by SquidGuard
            assert_eq!(log.field(field_dictionary::HTTP_RESPONSE_STATUS_CODE), Some(&SiemField::U64(503)));
            // Check that the URL was extracted correctly
            assert_eq!(log.field(field_dictionary::URL_DOMAIN), Some(&SiemField::from_str("hackpage.com")));
            // Check the port=80=http
            assert_eq!(log.field(field_dictionary::DESTINATION_PORT), Some(&SiemField::U64(80)));
            // Blocked means that the remote host could not send us information
            assert_eq!(log.field(field_dictionary::DESTINATION_BYTES), Some(&SiemField::U64(0)));
            // Check that the URL was categorized as Hacking
            assert_eq!(log.field(field_dictionary::RULE_CATEGORY), Some(&SiemField::from_str(WebProxyRuleCategory::Hacking.to_string())));
            // The method=GET but we could have tested POST
            assert_eq!(log.field(field_dictionary::HTTP_REQUEST_METHOD), Some(&SiemField::from_str("GET")));
            // Check that the URL path was extracted correctly
            assert_eq!(log.field(field_dictionary::URL_PATH), Some(&SiemField::from_str("/random-stuff/and-random.html")));
            // And also the URL query
            assert_eq!(log.field(field_dictionary::URL_QUERY), Some(&SiemField::from_str("?param_1=value_1&param_2=value_2")));
        },
        Err(_) => {
            // Exit with error if the log couldn't be parsed
            panic!("Cannot parse log")
        }
    }
}
```


### Conclusion

From this point onwards, we can start testing with real logs instead of syntetic ones. The opened possibilities are innumerable: from using real Windows logs (yeah Windows machines in Github actions) to testing multiple versions of the same product at once. 

For those payment products, with physical devices or that cannot be easily executed in CI / CD environments, it will be difficult to carry out integration tests. We will have to settle for synthetic logs. And yes, I am including in this category Firewalls (PaloAlto, SonicWall), Proxys (BlueCoat, Zscaler), VPNs (ASA, PulseSecure)...