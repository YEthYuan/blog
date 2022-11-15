---
title: Load Testing with Tsung
date: 2022-11-14 22:28:33
tags: [EN, UCSB, CS291A, Testing]
categories: Tools
---

# What should we observe during load testing?

- Response times
- Error rates
- Synthetic user success rate

We want to create load-test flows that resemble real traffic. There are a few things to consider:

- Ensuring flows contain a mixture of reads and writes
- Ensuring variance within the flows themselves (not all users have the same habits)
- The test framework should be able to utilize data returned from prior requests (e.g., create a submission and make a comment on it)

# Load Testing Tools

## High Performance Tools

- [apachebench](https://httpd.apache.org/docs/2.4/programs/ab.html) (ab)
- [Apache JMeter](https://jmeter.apache.org/)
- [httperf](https://github.com/httperf/httperf)
- [Tsung](http://tsung.erlang-projects.org/)

## Feature-rich Tools

- [Funkload](https://github.com/nuxeo/FunkLoad/)
- [Tsung](http://tsung.erlang-projects.org/)

## Others

- [k6](https://k6.io/docs/test-types/load-testing)
- [Locust](https://locust.io/)
- [Bees with Machine Guns](https://github.com/newsapps/beeswithmachineguns)

Tsung is a good choice as it is extremely configurable and delivers high performance.

# Tsung Usage

## Testing EC2 instances from EC2

We should the test instances in the same region as our elastic beanstalk deployment. This decision provides us with:

- Lower cost: AWS charges for bandwidth outside of the region
- Predictability: fewer moving parts between the test instance and the stack
- Less representative testing: none of our real users will be in the data center

The first two points are positive.

The third point could be a concern, but since we are essentially using these sort of tests for comparison between code changes, we're less concerned with being completely representative.

## Launching the Tsung Instance

Connect to the jump-box:

```shell
ssh -i TEAMNAME.pem TEAMNAME@ec2.cs291.com
```

Create an instance:

```shell
/bin/launch_tsung.sh
```

You can manually run the script commands in case you want to update any params.

```shell
cat /bin/launch_tsung.sh
```

The output will provide you with the ssh command to access the instance.

## The Tsung Instance

The home directory of the Tsung instance contains one file and a symlink:

```shell
[ec2-user@ip-10-140-200-96 ~]$ ls
tsung_example.xml
```

`tsung_example.xml` is a sample Tsung load-test configuration file that establishes various connections to [https://cs291.com](https://cs291.com/).

Run this Tsung test via:

```shell
tsung -f tsung_example.xml -k start
```

Once Tsung starts, visit the URL listed in the output.

## Tsung Server

While Tsung runs (and continues to run after testing with the `-k` option) it provides a simple web interface.

![Tsung Web Interface](https://yeyuan.pro/img/202211142255055.png)

## Tsung Interface

Out of the box, Tsung generates a report `/es/ts_web:report`, and a handful of graphs `/es/ts_web:graph`.

The server, and web-based access to these files ceases to be accessible once the Tsung process is stopped. Restarting Tsung only makes the graphs from the current run available via the web interface.

## Fetching Tsung results

Aside from fetching files through the web interface. You may want to obtain all of the Tsung results.

Assuming your Tsung instance IP address is `54.166.5.220` then the following command will synchronize the Tsung result files from all runs to your ec2.cs291.com account:

```shell
rsync -auvL ec2-user@54.166.5.220:tsung_logs .
```

Running that command a second time will only fetch any new files (assuming you run it from the same path). `rsync` is awesome!

## The Tsung XML File Structure

```html
<?xml version="1.0"?>
<!DOCTYPE tsung SYSTEM "/usr/local/share/tsung/tsung-1.0.dtd" [] >
<tsung loglevel="notice">

  <clients>...</clients>
  <servers>...</servers>
  <load>...</load>
  <options>...</options>
  <sessions>...</sessions>
  ...

</tsung>
```

For a more complete discussion please see Tsung's documentation: http://tsung.erlang-projects.org/user_manual/configuration.html

## Tsung Clients Section

Reference: http://tsung.erlang-projects.org/user_manual/conf-client-server.html

```html
<clients>
  <client host="localhost" use_controller_vm="true" maxusers='30000'/>
</clients>
```

The clients section specifies the clients to generate load from. You can use multiple tsung clients as part of a single test, and you may need to for your final load tests.

## Tsung Servers Section

Reference: http://tsung.erlang-projects.org/user_manual/conf-client-server.html

```html
<servers>
  <server host="cs291.com" port="443" type="ssl"></server>
</servers>
```

The servers section specifies where you intend to connect to. For your tests you will always want to modify the `host` value to match your deployment's domain.

## Tsung Load Section

Reference: http://tsung.erlang-projects.org/user_manual/conf-load.html

```html
<load>
  <arrivalphase phase="1" duration="60" unit="second">
    <users arrivalrate="2" unit="second"></users>
  </arrivalphase>
  <arrivalphase phase="2" duration="1" unit="minute">
    <users interarrival="2" unit="second"></users>
  </arrivalphase>
</load>
```

Here we describe two phases.

The first phase lasts 60 seconds where 2 new users arrive (`arrivalrate`) each second.

The second phase also lasts 60 seconds. Here 1 new user arrives (`interarrival`) every two seconds.

## Tsung Options Section

Reference: http://tsung.erlang-projects.org/user_manual/conf-options.html

```html
<options>
  <!-- Set connection timeout to 2 seconds -->
  <option name="global_ack_timeout" value="2000"></option>
</options>
```

## Tsung Sessions Section

Reference: http://tsung.erlang-projects.org/user_manual/conf-sessions.html

```html
<sessions>
  <session name="http-example" type="ts_http" weight="1">
    <request>
      <http method="GET" url="/" version="1.1"></http>
    </request>
    <thinktime random="true" value="2"></thinktime>
    <request>
      <http contents="user=foo" method="POST" url="/login" version="1.1"></http>
    </request>
  </session>
</sessions>
```

You can define multiple sessions and for each user one will be chosen according to the weights.

## Load Testing Your Server

- Modify the `server` tag's `host` section to point to your deployment's domain.
- Add a `request` tag for each resource fetched when you load your application's home page.
- Group requests to discrete *pages* in `<transaction>` if you want statistics on the individual pages.

## Obtaining Dynamic Variables

Reference: http://tsung.erlang-projects.org/user_manual/conf-advanced-features.html

Assume this HTML form:

```html
<form action="/create" method="POST">
  <input name="auth_token" type="hidden" value="gDTI=" />
</form>
```

The value `gDTI=` can be obtained from the response via:

```html
<request>
  <dyn_variable name="auth_token"></dyn_variable>
  <http method="GET" url="/new" version="1.1"></http>
</request>
```

## Using Dynamic Variables

We previously set a variable named `auth_token`. Let's use it.

```html
<request subst="true">
  <http content_type="application/x-www-form-urlencoded"
        contents="user=bboe&amp;auth_token=%%_auth_token%%"
        method="POST"
        url="/create"
        version="1.1">
  </http>
</request>
```

Substitution is done via `%%_VARIABLE_NAME%%`.

In rails CSRF protection values are usually stored in a hidden field named `authenticity_token`.

**Note**: The `&` character must be encoded like `&`. This causes a problem with the CSRF token.

## Other Dynamic Variables

The above example shows how to extract a variable from a form field. They can also be extracted from:

- HTML using [XPath](http://tsung.erlang-projects.org/user_manual/conf-advanced-features.html#xpath)
- arbitrary text using [Regexp](http://tsung.erlang-projects.org/user_manual/conf-advanced-features.html#regexp)
- JSON via [JSONPath](http://tsung.erlang-projects.org/user_manual/conf-advanced-features.html#jsonpath)

## Generating Random Variables

Variables can be explicitly set via a few functions and occur anywhere in a `<session>`. ([ref](http://tsung.erlang-projects.org/user_manual/conf-advanced-features.html#set-dynvars))

### Random Number as `rndint`

```html
<setdynvars sourcetype="random_number" start="3" end="32">
  <var name="rndint" />
</setdynvars>
```

### Random String as `rndstring1`

```html
<setdynvars sourcetype="random_string" length="13">
  <var name="rndstring1" />
</setdynvars>
```

For a more complete example please see: [Demo App's load_tests/critical.xml](https://github.com/scalableinternetservices/demo/blob/master/load_tests/critical.xml)

# Understanding Tsung's Output

## Tsung Output

- **connect**: The duration of the connection establishment
- **page**: Response time for each set of requests (a page is a group of requests not separated by a thinktime)
- **request**: Response time for each request
- **session**: The duration of a user's session

![Tsung Statistics](https://yeyuan.pro/img/202211142255056.png)

## Response Time Graph

![Tsung Response Time Graph](https://yeyuan.pro/img/202211142255057.png)

## Tsung HTTP Return Codes

Inspect the HTTP return code section:

- 2XX and 3XX status codes are good
- 4XX and 5XX status codes can indicate problems:
  - Too many requests?
  - Server-side bugs?
  - Bug in testing code?

Ideally you shouldn't see any 4XX and 5XX status codes in your tests unless you are certain they are due to exceeding the load on your web server's stack.

## Error Rate Graph

![Tsung Error Rate Graph](https://yeyuan.pro/img/202211142255058.png)

## Users Graph

![Tsung Users Graph](https://yeyuan.pro/img/202211142255059.png)

## Advice

### Reminder: Fetch your Tsung data as soon as it is available

```shell
rsync -auvL ec2-user@54.166.5.220:tsung_logs .
```

### Use tsplot to compare separate runs:

```shell
tsplot -d graphs m3_medium tsung_logs/20171108-2133/tsung.log m3_large tsung_logs/20171108-2146/tsung.log
```
