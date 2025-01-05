# Smuggler
[![contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat)](https://github.com/TheHermione/smuggler/issues)

![изображение](https://github.com/user-attachments/assets/c770f48c-a857-439d-97f5-ed9b867be236)

It's a little bit upgraded fork of an HTTP Request Smuggling/Desync testing [tool](https://github.com/defparam/smuggler) written in Python 3 by [@defparam](https://github.com/defparam/). 

## Few benefits were added:
- Telegram notifications
- Input file with urls
- Adding custom headers (cookie, bugbounty header, authorization header and etc.)

## IMPORTANT
This tool does not guarantee no false-positives or false-negatives. Just because a mutation may report OK does not mean there isn't a desync issue, but more importantly just because the tool indicates a potential desync issue does not mean there definitely exists one. The script may encounter request processors from large entities (i.e. Google/AWS/Yahoo/Akamai/etc..) that may show false positive results.
And use it only for educational purposes! I assume no responsibility :)

## Installation

```bash
git clone https://github.com/TheHermione/smuggler
cd smuggler
python3 smuggler.py -h
```

## Example Usage

Single host:
```bash
python3 smuggler.py -u <URL>
```

List of urls:
```bash
python3 smuggler.py -f <FILE_WITH_URLS>
```

## Options

```bash
usage: smuggler.py [-h] [-u URL] [-f FILE] [-v VHOST] [-x] [-m METHOD] [-l LOG] [-q] [-t TIMEOUT] [--no-color] [-H HEADER] [--cookie COOKIE] [--bb BB]
                   [-c CONFIGFILE] [-d DELAY]

options:
  -h, --help            show this help message and exit
  -u URL, --url URL     Target URL with endpoint
  -f FILE, --file FILE  Specify a file with urls
  -v VHOST, --vhost VHOST
                        Specify a virtual host
  -x, --exit_early      Exit scan on first finding
  -m METHOD, --method METHOD
                        HTTP method to use (e.g GET, POST) Default: POST
  -l LOG, --log LOG     Specify a log file
  -q, --quiet           Quiet mode will only log issues found
  -t TIMEOUT, --timeout TIMEOUT
                        Socket timeout value Default: 5
  --no-color            Suppress color codes
  -H HEADER, --header HEADER
                        Custom header to include in requests
  --cookie COOKIE       Custom cookie to include in requests
  --bb BB               BugBounty header to include in requests
  -c CONFIGFILE, --configfile CONFIGFILE
                        Filepath to the configuration file of payloads
  -d DELAY, --delay DELAY
                        Delay between requests in seconds
```

Smuggler at a minimum requires either a URL via the -u/--url argument or a list of URLs piped into the script via stdin.
If the URL specifies `https://` then Smuggler will connect to the host:port using SSL/TLS. If the URL specifies `http://`
then no SSL/TLS will be used at all. If only the host is specified, then the script will default to `https://`.

You have to set a `TELEGRAM_TOKEN` and `CHAT_ID` in `send_telegram_notification(text)` function if you wanna get vulnerability notofications during the scan:
```python
def send_telegram_notification(text):
	TELEGRAM_TOKEN = '876567809:uythgjiiuygjkuiyghjujygbnjyghbyffso'
	CHAT_ID = '1111111111'
	try:
		response = requests.post(
			f'https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage',
          		data={'chat_id': CHAT_ID, 'text': text}
      		)
		print('Notification sent successfully')
	except requests.RequestException as e:
		print(f'Error: {e}')
```

Use -v/--vhost \<host> to specify a different host header from the server address

Use -H/--header \<header: value> to include in requests

Use --cookie \<cookie_value> to include in requests

Use --bb \<bugbounty_header> to include in requests

Use -d/--delay \<float_value> to set a delay between requests in seconds

Use -x/--exit_early to exit the scan of a given server when a potential issue is found. In piped mode smuggler will just continue to the next host on the list

Use -m/--method \<method> to specify a different HTTP verb from POST (i.e GET/PUT/PATCH/OPTIONS/CONNECT/TRACE/DELETE/HEAD/etc...)

Use -l/--log \<file> to write output to file as well as stdout

Use -q/--quiet reduce verbosity and only log issues found

Use -t/--timeout \<value> to specify the socket timeout. The value should be high enough to conclude that the socket is hanging, but low enough to speed up testing (default: 5)

Use --no-color to suppress the output color codes printed to stdout (logs by default don't include color codes)

Use -c/--configfile \<configfile> to specify your smuggler mutation configuration file (default: default.py)

## Config Files
Configuration files are python files that exist in the ./config directory of smuggler. These files describe the content of the HTTP requests and the transfer-encoding mutations to test.


Here is example content of default.py:
```python
def render_template(gadget):
	RN = "\r\n"
	p = Payload()
	p.header  = "__METHOD__ __ENDPOINT__?alk=__RANDOM__ HTTP/1.1" + RN
	# p.header += "Transfer-Encoding: chunked" +RN	
	p.header += gadget + RN
	p.header += "Host: __HOST__" + RN
	p.header += "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.87 Safari/537.36" + RN
	p.header += "Content-type: application/x-www-form-urlencoded; charset=UTF-8" + RN
	p.header += "Content-Length: __REPLACE_CL__" + RN
	return p


mutations["nameprefix1"] = render_template(" Transfer-Encoding: chunked")
mutations["tabprefix1"] = render_template("Transfer-Encoding:\tchunked")
mutations["tabprefix2"] = render_template("Transfer-Encoding\t:\tchunked")
mutations["space1"] = render_template("Transfer-Encoding : chunked")

for i in [0x1,0x4,0x8,0x9,0xa,0xb,0xc,0xd,0x1F,0x20,0x7f,0xA0,0xFF]:
	mutations["midspace-%02x"%i] = render_template("Transfer-Encoding:%cchunked"%(i))
	mutations["postspace-%02x"%i] = render_template("Transfer-Encoding%c: chunked"%(i))
	mutations["prespace-%02x"%i] = render_template("%cTransfer-Encoding: chunked"%(i))
	mutations["endspace-%02x"%i] = render_template("Transfer-Encoding: chunked%c"%(i))
	mutations["xprespace-%02x"%i] = render_template("X: X%cTransfer-Encoding: chunked"%(i))
	mutations["endspacex-%02x"%i] = render_template("Transfer-Encoding: chunked%cX: X"%(i))
	mutations["rxprespace-%02x"%i] = render_template("X: X\r%cTransfer-Encoding: chunked"%(i))
	mutations["xnprespace-%02x"%i] = render_template("X: X%c\nTransfer-Encoding: chunked"%(i))
	mutations["endspacerx-%02x"%i] = render_template("Transfer-Encoding: chunked\r%cX: X"%(i))
	mutations["endspacexn-%02x"%i] = render_template("Transfer-Encoding: chunked%c\nX: X"%(i))
```

There are no input arguments yet on specifying your own customer headers and user-agents. It is recommended to create your own configuration file based on default.py and modify it to your liking.

Smuggler comes with 3 configuration files: default.py (fast), doubles.py (niche, slow), exhaustive.py (very slow)
default.py is the fastest because it contains less mutations.

specify configuration files using the -c/--configfile \<configfile> command line option

## Payloads Directory
Inside the Smuggler directory is the payloads directory. When Smuggler finds a potential CLTE or TECL desync issue, it will automatically dump a binary txt file of the problematic payload in the payloads directory. All payload filenames are annotated with the hostname, desync type and mutation type. Use these payloads to netcat directly to the server or to import into other analysis tools.
