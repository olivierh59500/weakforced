Weakforced
----------
The goal of 'wforce' is to detect brute forcing of passwords across many
servers, services and instances.  In order to support the real world, brute
force detection policy can be tailored to deal with "bulk, but legitimate"
users of your service, as well as botnet-wide slowscans of passwords.

The aim is to support the largest of installations, providing services to
hundreds of millions of users.  The current version of weakforced is not
quite there yet.

wforce is a project by Dovecot and Open-Xchange. For historical
reasons, it lives in the PowerDNS github tree. If you have any questions, email
neil.cook@open-xchange.com.

Here is how it works:
 * Report successful logins via JSON http-api
 * Report unsuccessful logins via JSON http-api
 * Query if a login should be allowed to proceed, should be delayed, or ignored via http-api

Runtime console for querying the status of logins, IP addresses, subnets.

wforce is aimed to receive message from services like:

 * IMAP
 * POP3
 * Webmail logins
 * FTP logins
 * Authenticated SMTP
 * Self-service logins
 * Password recovery services

By gathering failed and successful login attempts from as many services as
possible, brute forcing attacks can be detected and prevented more
effectively.

Inspiration:
http://www.techspot.com/news/58199-developer-reported-icloud-brute-force-password-hack-to-apple-nearly-six-month-ago.html

Installing
----------
From GitHub:

```
$ git clone https://github.com/PowerDNS/weakforced.git
$ cd weakforced
$ autoreconf -i
$ ./configure
$ make
```

This requires recent versions of libtool, automake and autoconf to be
installed.  Secondly, we require (versions tagged up to 1.0.0):
 * Recent g++ (4.8)
 * Boost 1.40+
 * Lua 5.1+ development libraries (or LuaJIT if you configure --with-luajit)
 * Getdns development libraries (if you want to use the DNS lookup functionality)
 * libsodium
 * python + virtualenv for regression testing
 * libgeoip-dev for GeoIP support
 * libsystemd-dev for systemd support
 * pandoc for building the manpages

The most recent versions also require:
 * Protobuf compiler and protobuf development libraries
 * libcurl-dev (OpenSSL version)
 * libhiredis-dev
 * libssl-dev
 * python-bottle for regression testing of webhooks

To build on OS X, `brew install readline gcc` and use
`./configure LDFLAGS=-L/usr/local/opt/readline/lib CPPFLAGS=-I/usr/local/opt/readline/include CC=gcc-5 CXX=g++-5 CPP=cpp-5`

Add --with-luajit to the end of the configure line if you want to use LuaJIT.

Policies
--------

There is a sensible default policy in wforce.conf (running without
this means *no* policy), and extensive support for crafting your own policies using
the insanely great Lua scripting language. Note that although there is
a single Lua configuration file, the report and allow functions run in
different lua states from the rest of the configuration, thus you
cannot share state.

Sample:

```
-- set up the things we want to track
field_map = {}
-- use hyperloglog to track cardinality of (failed) password attempts
field_map["diffFailedPasswords"] = "hll"
-- track those things over 6x10 minute windows
newStringStatsDB("OneHourDB", 600, 6, field_map)

-- this function counts interesting things when "report" is invoked
function twreport(lt)
	sdb = getStringStatsDB("OneHourDB")
	if (not lt.success)
	then
	   sdb:twAdd(lt.remote, "diffFailedPasswords", lt.pwhash)
	   addrlogin = lt.remote:tostring() .. lt.login
	   sdb:twAdd(addrlogin, "diffFailedPasswords", lt.pwhash)
	end
end

function allow(lt)
	sdb = getStringStatsDB("OneHourDB")
	if(sdb:twGet(lt.remote, "diffFailedPasswords") > 50)
	then
		return -1, "", "", {} -- BLOCK!
	end
	// concatenate the IP address and login string
	addrlogin = lt.remote:tostring() .. lt.login	
	if(sdb:twGet(addrlogin, "diffFailedPasswords") > 3)
	then
		return 3, "tarpitted", "diffFailedPasswords", {} -- must wait for 3 seconds
	end

	return 0, "", "", {} -- OK!
end
```

Many more metrics are available to base decisions on. Some example
code is in [wforce.conf](wforce.conf), and more extensive examples are
in [wforce.conf.example](wforce.conf.example).

To report (if you configured with 'webserver("127.0.0.1:8084", "secret")'):

```
$ for a in {1..101}
  do
    curl -X POST -H "Content-Type: application/json" --data '{"login":"ahu", "remote": "127.0.0.1", "pwhash":"1234'$a'", "success":"false"}' \
    http://127.0.0.1:8084/?command=report -u wforce:secret
  done
```

This reports 101 failed logins for one user, but with different password hashes.

Now to look up if we're still allowed in:

```
$ curl -X POST -H "Content-Type: application/json" --data '{"login":"ahu", "remote": "127.0.0.1", "pwhash":"1234"}' \
  http://127.0.0.1:8084/?command=allow -u wforce:super
{"status": -1, "msg": "diffFailedPasswords"}
```

It appears we are not!

You can also provide additional information for use by weakforce using
the optional "attrs" object. An example:

```
$ curl -X POST -H "Content-Type: application/json" --data '{"login":"ahu", "remote": "127.0.0.1",
"pwhash":"1234", "attrs":{"attr1":"val1", "attr2":"val2"}}' \
  http://127.0.0.1:8084/?command=allow -u wforce:super
{"status": 0, "msg": ""}
```

An example using the optional attrs object using multi-valued
attributes:

```
$ curl -X POST -H "Content-Type: application/json" --data '{"login":"ahu", "remote": "127.0.0.1",
"pwhash":"1234", "attrs":{"attr1":"val1", "attr2":["val2","val3"]}}' \
  http://127.0.0.1:8084/?command=allow -u wforce:super
{"status": 0, "msg": ""}
```

There is also a command to reset the stats for a given login and/or IP
Address, using the 'reset' command, the logic for which is also
implemented in Lua. The default configuration for reset is as follows:

```
function reset(type, login, ip)
	 sdb = getStringStatsDB("OneHourDB")
	 if (string.find(type, "ip"))
	 then
		sdb:twReset(ip)
	 end
	 if (string.find(type, "login"))
	 then
		sdb:twReset(login)
	 end
	 if (string.find(type, "ip") and string.find(type, "login"))
	 then
		iplogin = ip:tostring() .. login
		sdb:twReset(iplogin)
	 end
	 return true
end
```

To test it out, try the following to reset the login 'ahu':

```
$ curl -X POST -H "Content-Type: application/json" --data '{"login":"ahu"}'\
  http://127.0.0.1:8084/?command=reset -u wforce:super
{"status": "ok"}
```

You can reset IP addresses also:

```
$ curl -X POST -H "Content-Type: application/json" --data '{"ip":"128.243.21.16"}'\
  http://127.0.0.1:8084/?command=reset -u wforce:super
{"status": "ok"}
```

Or both in the same command (this helps if you are tracking stats using compound keys
combining both IP address and login):

```
$ curl -X POST -H "Content-Type: application/json" --data '{"login":"ahu", "ip":"FE80::0202:B3FF:FE1E:8329"}'\
  http://127.0.0.1:8084/?command=reset -u wforce:super
{"status": "ok"}
```

Finally there is a "ping" command, to check the server is up and
answering requests:

```
$ curl -X GET http://127.0.0.1:8084/?command=ping -u wforce:super
{"status": "ok"}
```

Console
-------
Available over TCP/IP, like this:
```
setKey("Ay9KXgU3g4ygK+qWT0Ut4gH8PPz02gbtPeXWPdjD0HE=")
controlSocket("0.0.0.0:4004")
```

Launch wforce as a daemon (`wforce --daemon`), to connect, run `wforce -c`.
Comes with autocomplete and command history. If you put an actual IP address
in place of 0.0.0.0, you can use the same config to listen and connect
remotely.

To get some stats, try:
```
> stats()
40 reports, 8 allow-queries, 40 entries in database
```

Spec
----
We report 4 mandatory fields plus one optional field in the LoginTuple

 * login (string): the user name or number or whatever
 * remote (ip address): the address the user arrived on
 * pwhash (string): a highly truncated hash of the password used
 * success (boolean): was the login a success or not?
 * attrs (json object): additional information about the login. For example, attributes from a user database.

All are rather clear, but pwhash deserves some clarification. In order to
distinguish actual brute forcing of a password, and repeated incorrect but
identical login attempts, we need some marker that tells us if passwords are
different.

Naively, we could hash the password, but this would spread knowledge of
secret credentials beyond where it should reasonably be. Even if we salt and
iterate the hash, or use a specific 'slow' hash, we're still spreading
knowledge.

However, if we take any kind of hash and truncate it severely, for example
to 12 bits, the hash tells us very little about the password itself - since
one in 4096 random strings will match it anyhow. But for detecting multiple
identical logins, it is good enough.

For additional security, hash the login name together with the password - this
prevents detecting different logins that might have the same password.

NOTE: wforce does not require any specific kind of hashing scheme, but it
is important that all services reporting successful/failed logins use the
same scheme!

When in doubt, try:

```
TRUNCATE(SHA256(SECRET + LOGIN + '\x00' + PASSWORD), 12)
```

Which denotes to take the first 12 bits of the hash of the concatenation of
a secret, the login, a 0 byte and the password.  Prepend 4 0 bits to get
something that can be expressed as two bytes.

API Calls
---------
We can call 'report', and 'allow' commands. The optional 'attrs' field
enables the client to send additional data to weakforced.

To report, POST to /?command=report a JSON object with fields from the
LoginTuple as described above.

To request if a login should be allowed, POST to /?command=allow, again with
the LoginTuple. The result is a JSON object with a "status" field. If this is -1, do
not perform login validation (i.e. provide no clue to the client if the password
was correct or not, or even if the account exists).

If 0, allow login validation to proceed. If a positive number, sleep this
many seconds until allowing login validation to proceed.

Load balancing: siblings
------------------------
For high-availability or performance reasons it may be desireable to run
multiple instances of wforce. To present a unified view of status however,
these instances then need to share the login tuples. To do so, wforce
implements a simple knowledge-sharing system.

Tuples received are broadcast (best effort, UDP) to all siblings. The
sibling list is parsed such that we don't broadcast messages to ourselves
accidentally, and can thus be identical across all servers.

To define siblings, use:

```
setKey("Ay9KXgU3g4ygK+qWT0Ut4gH8PPz02gbtPeXWPdjD0HE=")
addSibling("192.168.1.79")
addSibling("192.168.1.30")
addSibling("192.168.1.54")
siblingListener("0.0.0.0")
```

The first line sets the authentication and encryption key for our sibling
communications. To make your own key (recommended), run `makeKey()` on the
console and paste the output in all your configuration files.

This last line configures that we also listen to our other siblings (which
is nice).  The default port is 4001, the protocol is UDP.

To view sibling stats:

```
> siblings()
Address                             Sucesses  Failures     Note
192.168.1.79:4001                   18        7
192.168.1.30:4001                   25        0
192.168.1.54:4001                   0         0            Self
```

With this setup, several wforces are all kept in sync, and can be load
balanced behind (for example) haproxy, which incidentally can also offer SSL.
