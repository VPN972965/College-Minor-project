import argparse
import requests
import urllib3
import re
import sys
import time
from urllib.parse import urlparse, parse_qs, urlencode, unquote, quote

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Define global variables
concurrency = 50
verbose = False
output_file = ''
payload = 'V'
user_agent = 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36'
proxy = ''
request_data = ''
method = 'GET'
custom_headers = []

def banner():
    print("""
 __      __
\ \    / /
 \ \  / / 
  \ \/ /  
   \  /   
    \/    
                         
    1.0 Mr_ViP.iN
    """)

def parse_arguments():
    global concurrency, verbose, output_file, payload, user_agent, proxy, request_data, custom_headers
    
    parser = argparse.ArgumentParser()
    parser.add_argument('-c', type=int, default=50, help='Set the Concurrency')
    parser.add_argument('-v', action='store_true', help='Verbose mode')
    parser.add_argument('-p', type=str, default='V', help='Payload you want to send to check reflection')
    parser.add_argument('-o', type=str, default='', help='Save result to output file')
    parser.add_argument('-d', type=str, default='', help='Request data for POST based reflection testing')
    parser.add_argument('-x', type=str, default='', help='Proxy URL. Example: http://127.0.0.1:8080')
    parser.add_argument('-u', type=str, default=user_agent, help='Set custom user agent. Default is Mozilla')
    parser.add_argument('-H', '--header', action='append', default=[], help='Set custom headers.')

    args = parser.parse_args()

    concurrency = args.c
    verbose = args.v
    output_file = args.o
    payload = args.p
    user_agent = args.u
    proxy = args.x
    request_data = args.d
    custom_headers = args.header

    if verbose:
        banner()

def main():
    parse_arguments()

    if payload:
        if output_file:
            with open(output_file, 'w') as f:
                pass
            print(f"Created {output_file}")

        for _ in range(concurrency):
            test_reflection(payload, verbose, output_file, request_data)
    else:
        print("No payload provided.")
        sys.exit(1)

    if verbose:
        print("\nFinished checking, thank you for using V.")

def test_reflection(payload, verbose, output_file, request_data):
    time.sleep(0.5)
    for link in sys.stdin:
        link = link.strip()
        check_reflection(link)

def check_reflection(link):
    global request_data, method

    decoded = unquote(link)
    try:
        u = urlparse(decoded)
    except Exception as e:
        try:
            decoded = quote(link)
            u = urlparse(decoded)
        except Exception as e:
            print(f"Error: {e}")
            return

    if verbose:
        print(f"[+] Testing URL: {link}")

    q = parse_qs(u.query)
    if request_data:
        method = "POST"
        q = parse_qs(request_data)
    else:
        method = "GET"

    for key, value in q.items():
        tm = value[0]
        q[key] = [payload]
        if method == "GET":
            new_query = urlencode(q, doseq=True)
            new_url = u._replace(query=new_query).geturl()
        elif method == "POST":
            request_data = urlencode(q, doseq=True)
            new_url = u.geturl()

        _, body, _ = request_func(new_url, request_data, method)

        if re.search(payload, body):
            if verbose:
                print(f"URL: {new_url}")
                print(f"Reflected Param: {key}")
            else:
                print(f"{new_url}\n")

            if output_file:
                with open(output_file, 'a') as f:
                    f.write(f"{new_url}\n")

        q[key] = [tm]

def request_func(url, data, method):
    headers = {
        'User-Agent': user_agent
    }
    if method == 'POST':
        headers['Content-Type'] = 'application/x-www-form-urlencoded'
    
    for header in custom_headers:
        key, value = header.split(':', 1)
        headers[key.strip()] = value.strip()

    proxies = {}
    if proxy:
        proxies = {
            'http': proxy,
            'https': proxy,
        }

    if verbose:
        print(f"Requesting URL: {url}")
        print(f"Headers: {headers}")
        print(f"Data: {data}")

    try:
        if method == 'GET':
            response = requests.get(url, headers=headers, proxies=proxies, verify=False, allow_redirects=False)
        else:
            response = requests.post(url, data=data, headers=headers, proxies=proxies, verify=False, allow_redirects=False)
        
        return response, response.text, None
    except requests.RequestException as e:
        print(f"Request failed: {e}")
        return None, '', [e]

if __name__ == "__main__":
    main()
