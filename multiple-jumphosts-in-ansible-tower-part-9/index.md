# Multiple Jumphosts in Ansible Tower - Part 9

## Pull secrets from remote 3rd-party vaults over multiple jumphost hops and use the credentials to connect to target hosts

[Alexei.Karve](https://developer.ibm.com/recipes/author/karve/)

Tags: Cloud computing, DevOps, Linux

Published on August 7, 2020 / Updated on August 19, 2020

![](images/Screen-Shot-2020-08-06-at-6.34.47-PM-1024x423.png)

### Overview

Skill Level: Advanced

Ansible Tower, Hashicorp Vault, Cyberark Conjur

This recipe will show how to retrieve ssh credentials from a remote Hashicorp Vault or Cyberark Conjur over multiple jumphosts and use them to connect to target hosts.

### Ingredients

Ansible Tower/AWX, Hashicorp Vault, Cyberark Conjur

### Step-by-step

#### 1. Introduction

Users upload credentials to Tower for access to machines and external services. Machine Credentials are utilized by Tower for authentication when launching Jobs against machines. Tower encrypts passwords and key information in the Tower database and never makes secret information visible via the API. Ansible Tower uses SSH to connect to remote Linux hosts after decrypting the credentials.

With external credentials backed by credential plugins, credential fields like an SSH Private key with optional passphrase can be mapped to values stored in a secret management system instead of providing them to Tower directly. Ansible Tower links the target credential fields to an external system credential that can be set by clicking on the search icon on the fields when creating the required target credentials. These external secret values will be fetched prior to running a playbook that needs them. Tower provides a credential plugin interface to add new external credential types to Tower so that it can be extended to support other secret management systems. This mechanism however requires direct access from Tower to the third-party credentials.

In this Part 9, we provide mechanisms for accessing the credentials in the remote vault over multiple jumphosts and using them to connect to multiple target host endpoints. The goal in this part is to show the approach, not expose all options for the 3rd party vault. You can extend and customize the code once you understand the technique of calling the vault REST API. An example with setting up of Hashicorp Vault is provided in Section 2 and another with CyberArk Conjur in Section 8.

#### 2. Example setup with the Hashicorp Vault

We will initially install a development instance of the Hashicorp Vault in Docker. Start with installing/starting iptables and docker if not already done.

`yum install iptables-services`\
`systemctl enable iptables`\
`systemctl start iptables`\
`systemctl stop docker`\
`systemctl start docker`

These instructions will use an instance of the vault installed on a jumphost ec2-52-201-237-93.compute-1.amazonaws.com.

docker run -d --cap-add=IPC_LOCK -e **'VAULT_DEV_ROOT_TOKEN_ID=myroot'** -e VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:1234' **-p1234**:1234 vault

The above command will start a development instance of the vault. Get the containerid and check the docker logs to find the Unseal key and root token.

```
docker logs $containerid

==> Vault server configuration:

             Api Address: http://0.0.0.0:1234
                     Cgo: disabled
         Cluster Address: https://0.0.0.0:1235
              Go Version: go1.14.4
              Listener 1: tcp (addr: "0.0.0.0:1234", cluster address: "0.0.0.0:1235", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.5.0

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory\
and starts unsealed with a single unseal key. The root token is already\
authenticated to the CLI, so you can immediately begin using Vault.

Set the following environment variable:

    $ export VAULT_ADDR='http://0.0.0.0:1234'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: bxbhvoaj/Bck8NU47PZESZmxTQRsT/ciVTC6iEkxu5k=
**Root Token: myroot**

Development mode should NOT be used in production installations!

==> Vault server started! Log data will stream in below:
```
Run the docker inspect command to find the IP address of the container that we will use later.

`docker inspect $containerid | grep IPAddress | tail -1`

Output:

`"IPAddress": "172.17.0.3",`

As a sidenote, if we want to find the curl commands that we can use to interact with the vault, we can use the **-output-curl-string** as shown below. This will work after we create the testkv engine on the vault.

`docker exec -it $containerid sh`

`/ # export VAULT_TOKEN=myroot`\
`/ # export VAULT_ADDR=http://127.0.0.1:1234`\
`/ # vault kv get -output-curl-string /testkv/testsecrets/jumphost`

`curl -H "X-Vault-Request: true" -H "X-Vault-Token: $(vault print token)" http://127.0.0.1:1234/v1/testkv/data/testsecrets/jumphost`

Getting these curl commands generated for the vault commands helps with the API we need to call to add and retrieve credentails.

#### 3. Connecting to the vault using a ssh tunnel

Now that we have an instance of the vault running, we want to connect to this vault from our Laptop/Tower by creating an ssh connection to the jumphost. Our Laptop has ssh access to this jumphost but not to other ports. Our goal is to add and then retrieve the ssh host endpoint credentials from the vault for connecting to the host endpoints. We create a single hop tunnel from Laptop(/Tower) to the jumphost ec2-52-201-237-93.compute-1.amazonaws.com listening on 127.0.0.1:1234

`ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -CfNq  -D 127.0.0.1:1234 -p 22 -i ~/amazontestkey.pem ec2-user@ec2-52-201-237-93.compute-1.amazonaws.com -v`

Start Google Chrome (on Mac) using the socks5 tunnel

`"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --user-data-dir="$HOME/proxy-profile" --proxy-server="socks5://localhost:1234"`

Connect to docker container ip address that was ouput with the docker inspect command previously http://172.17.0.3:1234/ui

Click on Enable new engine and create "testkv" as shown in screenshot below.

![](images/Screen-Shot-2020-08-03-at-8.58.47-AM.png)

Click on Enable Engine to see the testkv get created.

![](images/Screen-Shot-2020-08-02-at-12.32.59-PM.png)

This showed how to connect to the vault using a browser over the tunnel. Next we will call the curl commands from python to write credentials to the testkv.

You will run the following python code on your Laptop to add keyvalue to vault. Note the vault_url = 'http://172.17.0.3:1234/v1/' + secret_path and the curl proxy using socks5h://localhost:1234.

**importcreds.py**
``` python
import yaml
import json
import subprocess

def read_creds(filePath):
    with open(filePath) as f:
        creds_data = yaml.safe_load(f) # json.load() -- for reading json file
    print ("finish read creds")
    return creds_data["all_creds"]

def feed_creds(creds_data):
    secret_path = 'testkv/data/testsecrets/'
    vault_url = 'http://172.17.0.3:1234/v1/' + secret_path
    app_token = 'myroot'
    authorization = 'X-Vault-Token: ' + app_token
    for cred in creds_data:
        query_url = vault_url + cred["secret_name"]
        content = {}
        content['data'] = cred["data"]
        print(json.dumps(content))
        cmd = ['curl','--proxy','socks5h://localhost:1234','-X','POST','-k','-H',authorization,query_url,'-d',json.dumps(content) ]
        #print(cmd)
        try:
            proc = subprocess.Popen(cmd, stdout=subprocess.PIPE)
            out,err = proc.communicate()
            print (out)
        except Exception as err:
            print (err)
def main():
    filePath='./sample_creds.yaml'
    creds_data = read_creds(filePath)
    feed_creds(creds_data)
if __name__ == '__main__':
    main()
```

Create a sample_creds.yaml with the base64 encoded values for the keys and passphrase (for your endpoints and/or jumphosts) and add them to the sample_creds.yaml below. The ssh_user is plain text.

`ssh-keygen -f ./alexei-key-ecdsa-jumphost -t ecdsa -b 521`\
`ssh-keygen -f ./alexei-key-ecdsa-aakrhel001 -t ecdsa -b 521`\
`...`

`echo -n Passphrase4ec2 | base64`\
`cat alexei-key-ecdsa-jumphost | base64`\
`echo -n Passphrase4aakrhel001 | base64`\
`cat alexei-key-ecdsa-aakrhel001 | base64`\
`...`

**sample_creds.yaml**

``` yaml
all_creds:
  - secret_name: jumphost
    data:
      ssh_user: ec2-user
      ssh_private_key: <base64 encoded ssh private key of jumphost>
      ssh_passphrase: <base64 encoded passphrase of jumphost>
  - secret_name: aakrhel001
    data:
      ssh_user: ec2-user
      ssh_private_key: <base64 encoded ssh private key of aakrhel001>
      ssh_passphrase: <base64 encoded passphrase of aakrhel001>
…
```

Run the python code:

`python import_creds.py`

This will create the credentials in the vault as shown in screenshot below that you can see in your browser.

![](images/Screen-Shot-2020-08-03-at-9.16.37-AM.png)

Specifically, the secret for aakrhel001 looks as follows:

![](images/Screen-Shot-2020-08-02-at-12.31.30-PM.png)

**Retrieving the credential**

Now that the credentials are added, lets run the curl command directly to retrieve the keyvalues for the "jumphost" credential. Notice the /data that is inserted after the engine name for v2 engine we created.

`curl --proxy socks5h://localhost:1234 -H "X-Vault-Token: myroot" 'http://172.17.0.3:1234/v1/testkv/data/testsecrets/jumphost'`

`{"request_id":"a6a7c6bf-4d0d-803f-c27a-07f2c59a4748","lease_id":"","renewable":false,"lease_duration":0,"data":{"data":{"<base64 encoded passphrase of jumphost>":"","ssh_private_key":"<base64 encoded ssh private key of jumphost>","ssh_user":"ec2-user"},"metadata":{"created_time":"2020-08-03T13:15:25.744178792Z","deletion_time":"","destroyed":false,"version":2}},"wrap_info":null,"warnings":null,"auth":null}`

**Retrieve the passphrase**

`curl --proxy socks5h://localhost:1234 -H "X-Vault-Token: myroot" 'http://172.17.0.3:1234/v1/testkv/data/testsecrets/jumphost' 2>/dev/null | jq --raw-output .data.data.ssh_passphrase | base64 -d`

`<passphrase>`

**Retrieve the ssh private key**

`curl --proxy socks5h://localhost:1234 -H "X-Vault-Token: myroot" 'http://172.17.0.3:1234/v1/testkv/data/testsecrets/jumphost' | jq --raw-output .data.data.ssh_private_key | base64 -d`

`-----BEGIN OPENSSH PRIVATE KEY-----`\
`<private key data>`\
`-----END OPENSSH PRIVATE KEY-----`

**Python code that calls curl to retrieve secrets**

``` python
import yaml
import json
import subprocess
def main():
    secret_path = 'testkv/data/testsecrets/'
    vault_url = 'http://172.17.0.3:1234/v1/' + secret_path
    app_token = 'myroot'
    authorization = 'X-Vault-Token: ' + app_token
    query_url = vault_url + "jumphost"
    cmd = ['curl','--proxy','socks5h://localhost:1234','-k' ,'-H', authorization ,'-H','Content-type: application/json','-X' ,'GET' ,query_url ]
    print (cmd)
    try:
        proc = subprocess.Popen(cmd, stdout=subprocess.PIPE)
        out,err = proc.communicate()
        print (out)
    except Exception as err:
        print (err)
if __name__ == '__main__':
    main()
```

The output from the python code is shown below:

`python export_creds.py`

`{"request_id":"07853bd9-9aa1-1b27-c24e-52f2125ae459","lease_id":"","renewable":false,"lease_duration":0,"data":{"data":{"ssh_passphrase":"<base64 encoded passphrase of jumphost>","ssh_private_key":"<base64 encoded ssh private key of jumphost>","ssh_user":"ec2-user"},"metadata":{"created_time":"2020-08-03T13:15:25.744178792Z","deletion_time":"","destroyed":false,"version":2}},"wrap_info":null,"warnings":null,"auth":null}`

#### 4. Using the Unix domain socket file to retrieve remote vault credentials

The curl command does not allow using of the unix socket file as a socks5 proxy. The curl --unix-socket option was added in curl 7.40. However, it sends a straight HTTP request, and makes no attempt to negotiate SOCKS, so curl's --unix-socket option won't be suitable for use with socks tunnel. This is shown with output below:

**Create a unix domain socket /tmp/mysock.sock2**

`python`\
`>>> import socket`\
`>>> sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)`\
`>>> sock.bind('/tmp/mysock.sock2')`\
`>>> sock.listen(1)`\
`>>> connection, client_address = sock.accept()`

**Send a curl request using the --unix-socket option**

`curl --unix-socket /tmp/mysock.sock2 http://aakrhel001.yellowykt.com:1234/v1/testkv/data/testsecrets/aakrhel001`

**and back in python, check what it sent**

`>>> req = connection.recv(4096)`\
`>>> req`\
`'GET /v1/testkv/data/testsecrets/aakrhel001 HTTP/1.1\r\nHost: aakrhel001.yellowykt.com:1234\r\nUser-Agent: curl/7.64.1\r\nAccept: */*\r\n\r\n'`\
`>>>`

Thus curl's --unix-socket assumes an HTTP proxy and it doesn't speak SOCKS. The "curl --socks5" works with socks port. We want this mechanism to work with unix domain socket file, we cannot use curl, but will have to use python code.

We have another instance of vault created exactly the same way as above on aakrhel001.yellowykt.com. The jumphost has access to aakrhel001.yellowykt.com on port 1234. If we replace the 172.17.0.3 with the aakrhel001.yellowykt.com, it works exactly the same way allowing us to add and retrieve the credentials for the vault over the jumphost. Instead of the single jumphost, we can create the tunnel over multiple jumphosts and that will allow us to retrieve credentials from a vault accessible over multiple hops of jumphosts. For the rest of the sections, we will use this vault instance on aakrhel001.yellowykt.com because this is accessible from connections, we have been using for the 5 jumphost hops (that we will use in later sections).

The following python code shows how to retrieve the credentials using socket instead of curl. Two options are shown. For the socks tunnel we create on localhost:1234, we use the uncommented **Option 1**.

``` python
import paramiko
import socks

# Option 1: Use Socks Port
s=socks.create_connection(("aakrhel001.yellowykt.com",1234), timeout=5, proxy_type=socks.SOCKS5, proxy_addr="127.0.0.1", proxy_port=1234)

# Option 2: Use Unix Socket File
# s=socks.create_connection(("aakrhel001.yellowykt.com",1234), timeout=5, proxy_type=socks.SOCKS5, proxy_addr="/tmp/mysock.sock")

# send
request = "GET /v1/testkv/data/testsecrets/jumphost HTTP/1.1\r\nX-Vault-Token: myroot\r\nHost:%s\r\n\r\n" % "aakrhel001.yellowykt.com"
s.send(request.encode())  
# receive
response = s.recv(4096)  
http_response = repr(response)
http_response_len = len(http_response)
print(http_response_len)
print(http_response)
s.close()

print(http_response.split("\\r\\n\\r\\n")[0]) # headers
print(http_response.split("\\r\\n\\r\\n")[1]) # body
```
This will retrieve the credentials from the remote vault over the tunnel and print them.

If we were to create the tunnel that uses the unix socket file /tmp/mysock.sock (instead of the socket port) to connect to the jumphost ec2-52-201-237-93.compute-1.amazonaws.com as follows, we can uncomment the **Option 2** from above. Note that this requires the modified python urllib3 and PySocks as was shown previously in [Part 5](..//multiple-jumphosts-in-ansible-tower-part-5/ "Multiple Jumphosts in Ansible Tower - Part 5").

`ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -CfNq -D /tmp/mysock.sock -p 22 -i ~/amazontestkey.pem ec2-user@ec2-52-201-237-93.compute-1.amazonaws.com -v`

So now we can use either the socket port or the socket file. This python code shown above outputs the raw http output that still needs to be parsed.

Another easier way is to use the requests module that is an easier to use HTTP library in python.

``` python
import requests
import base64
import json

# Option 1: Use Socks Port
r=requests.get("http://aakrhel001.yellowykt.com:1234/v1/testkv/data/testsecrets/jumphost", proxies={"http": "socks5h://127.0.0.1:1234", "https": "socks5h://127.0.0.1:1234"},headers={"X-Vault-Token":"myroot"})

# Option 2: Use Unix Socket File
r=requests.get("http://aakrhel001.yellowykt.com:1234/v1/testkv/data/testsecrets/jumphost", proxies={"http": "socks5h://unixsocket/tmp/mysock.sock", "https": "socks5h://unixsocket/tmp/mysock.sock"},headers={"X-Vault-Token":"myroot"})
r.content
j=json.loads(r.content.replace(b'\n',b''))
base64.b64decode(j['data']['data']['ssh_passphrase'])
base64.b64decode(j['data']['data']['ssh_private_key'])
```

The above code directly allows us to retrieve the ssh_private_key and the ssh_passphrase.

We can use the hvac python library to retrieve the credentials from the vault using the proxy. The following shows the output when using the socket port.

`python`\
`>>> import hvac`\
`>>> client = hvac.Client(url='http://172.17.0.3:1234', token='myroot',proxies={"http": "socks5h://127.0.0.1:1234", "https": "socks5h://127.0.0.1:1234"})`\
`>>> res = client.is_authenticated()`\
`>>> print("res:", res)`\
`('res:', True)`\
`>>> jumphost_secret = client.read('testkv/data/testsecrets/jumphost')`\
`>>> print(jumphost_secret)`\
`{u'lease_id': u'', u'warnings': None, u'wrap_info': None, u'auth': None, u'lease_duration': 0, u'request_id': u'44dd0de3-5625-a047-c666-4511af4560df', u'data': {u'data': {u'ssh_passphrase': u'<base64 encoded passphrase of jumphost>', u'ssh_user': u'ec2-user', u'ssh_private_key': '":"<base64 encoded ssh private key of jumphost>'}, u'metadata': {u'created_time': u'2020-08-03T13:15:25.744178792Z', u'destroyed': False, u'version': 2, u'deletion_time': u''}}, u'renewable': False}`\
`>>>`

This method with the hvac library also works with the modified pysocks if using the unix domain socket file /tmp/mysock.sock

`python`\
`>>> import hvac`\
`>>> client = hvac.Client(url='http://172.17.0.3:1234', token='myroot',proxies={"http": "socks5h://unixsocket/tmp/mysock.sock", "https": "socks5h://unixsocket/tmp/mysock.sock"})`\
`>>> res = client.is_authenticated()`\
`>>> res`\
`True`\
`>>> jumphost_secret = client.read('testkv/data/testsecrets/jumphost')`\
`>>> print(jumphost_secret)`\
`{u'lease_id': u'', u'warnings': None, u'wrap_info': None, u'auth': None, u'lease_duration': 0, u'request_id': u'c1938439-af00-a669-4694-9a655ee22584', u'data': {u'data': {u'ssh_passphrase': u'<base64 encoded passphrase of jumphost>', u'ssh_user': u'ec2-user', u'ssh_private_key': u'<base64 encoded ssh private key of jumphost>'}, u'metadata': {u'created_time': u'2020-08-03T13:15:25.744178792Z', u'destroyed': False, u'version': 2, u'deletion_time': u''}}, u'renewable': False}`\
`>>>`

#### 5. Using the Ansible lookup plugin

We use the lookup plugin hashi_vault if our Laptop/Tower has direct connectivity to the vault. However, hashi_vault does not allow connecting to socks proxy. We can make a minor change to enable proxies in the [hashi_vault plugin](https://raw.githubusercontent.com/ansible/ansible/stable-2.9/lib/ansible/plugins/lookup/hashi_vault.py "Original hashi_vault plugin code") as shown in [lookup_plugins/hashi_vault2.py](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/lookup_plugins/hashi_vault2.py "hashi_vault2.py"). We assume that you have created a tunnel to the jumphost from which the vault (on aakrhel001.yellowykt.com:1234) is accessible as shown in previous section using socket port localhost:1234.

**retrieve_credentials_playbook1.yaml**

``` yaml
- hosts: localhost
  gather_facts: no
  vars:
    token: myroot
    vault_secret_path: /testkv/data/testsecrets/jumphost/
    keys: ['ssh_passphrase','ssh_private_key']
    vault_url: http://aakrhel001.yellowykt.com:1234
    contents: "{{ lookup('file_content', '/tmp/foo.txt') }}"
  tasks:
    - debug:
        msg: "{{ lookup('hashi_vault2', 'secret={{ vault_secret_path }} token={{ token }} url={{ vault_url }}', errors='ignore').data[item] | b64decode }}"
        msg: "{{ lookup('hashi_vault2', 'secret={{ vault_secret_path }} token={{ token }} url={{ vault_url }}', errors='ignore').data[item] | b64decode }}"
      with_items: "{{ keys }}"
```

and run it from the Mac:

`export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES # On Mac to prevent error from security to restrict multithreading`\
`export VAULT_PROXY='{"http": "socks5h://localhost:1234", "https": "socks5h://localhost:1234"}'`

`ansible-playbook -i "localhost," retrieve_credentials_playbook1.yaml`

We can also do this as follows using the lookup with ":data". **The lookup plugin does not execute until a task references the variable** cred using the lookup. Even the myuser, mypassphrase and mysshprivatekey run a lookup only when referenced in a task. Consequently, each reference to these variables cred, myuser, mypassphrase and mysshprivatekey results in a separate lookup call over the multiple jumphost hops to the vault every time they are used in a task.

**retrieve_credentials_playbook1_new.yaml**
``` yaml
- hosts: localhost
  connection: local
  gather_facts: no
  vars:
    token: myroot
    keyname: jumphost
    vault_secret_path: /testkv/data/testsecrets/{{ keyname }}
    vault_url: http://aakrhel001.yellowykt.com:1234
    jh_socks_port: socks5h://127.0.0.1:1234
    cred: "{{ lookup('hashi_vault2', 'secret='+vault_secret_path+':data token='+token+' url='+vault_url+' proxies={\"http\":\"'+jh_socks_port+'\",\"https:\":\"'+jh_socks_port+'\"}', errors='ignore') }}"
    myuser: "{{ cred.ssh_user }}"
    mypassphrase: "{{ cred.ssh_passphrase|b64decode }}"
    mysshprivatekey:  "{{ cred.ssh_private_key|b64decode }}"
    #myuser: "{{ lookup('hashi_vault2', 'secret='+vault_secret_path+':data token='+token+' url='+vault_url, errors='ignore').ssh_user }}"
    #mypassphrase: "{{ lookup('hashi_vault2', 'secret='+vault_secret_path+':data token='+token+' url='+vault_url, errors='ignore').ssh_passphrase|b64decode }}"
    #mysshprivatekey: "{{ lookup('hashi_vault2', 'secret='+vault_secret_path+':data token='+token+' url='+vault_url, errors='ignore').ssh_private_key|b64decode }}"
  tasks:
    - debug:
        msg:
          - "cred: {{ cred }}"
    - debug:
        msg:
          - "myuser: {{ myuser }}"
          - "mypassphrase: {{ mypassphrase }}"
          - "mysshprivatekey: {{ mysshprivatekey }}"
```

The VAULT_PROXY is exported before running the ansible-playbook. You can export the VAULT_PROXY that uses the socks5h://unixsocket/tmp/mysock.sock if you have created the tunnel using the unix domain socket file.

We cannot use the environment keyword in the play on local environment that lookup runs under. The environment keyword only affects remote task execution (invoked inside connection). If you try to set the environment in a shell process, the shell process has a copy of the parent's environment and no access to the parent process's environment. When the shell process terminates, any changes made to its environment are lost. The lookup plugin does not see the added environment variable. We need to be able to pass the proxies as a parameter and retrieve the key into a file. This can be done with the [following changes](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/lookup_plugins/hashi_vault2.py#L167-L168 "Changes for proxies") to the file hashi_vault.py. The os.environ.get('VAULT_PROXY') provides the default for the kwargs.get if the argument does not have the proxies.

```
        self.proxies = kwargs.get('proxies', os.environ.get('VAULT_PROXY'))
        if self.proxies is not None: self.proxies=json.loads(self.proxies)
```

and adding the proxies=self.proxies to the hvac. Client call are in [4 places](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/lookup_plugins/hashi_vault2.py#L195-L198 "Changes to use proxies") in hashi_vault.py

```
                    self.client = hvac.Client(...,proxies=self.proxies)
```

We can test it with the following playbook. The "proxies=" should not have any spaces in the argument. Ansible tries to convert the proxies to an object (and adds spaces). We therefore need to create the proxies_str as a string with no spaces.

**retrieve_credentials_playbook2.yaml**

``` yaml
- hosts: localhost
  gather_facts: no
  connection: local
  vars:
    token: myroot
    vault_secret_path: /testkv/data/testsecrets/jumphost/
    vault_url: http://aakrhel001.yellowykt.com:1234
    jh_socks_port: socks5h://localhost:1234
    proxies: {"http":"{{ jh1_socks_port if jh1_socks_port is defined else jh_socks_port }}","https:":"{{ jh1_socks_port if jh1_socks_port is defined else jh_socks_port }}"}
  tasks:
    - set_fact:
        proxies_str: '{{ proxies | to_json | replace(" ","") | string }}'
    - debug:
        msg:
          - "vault_url: {{ vault_url }}"
          - "proxies: {{ proxies_str }}"
          - "vault_secret_path: {{ vault_secret_path }}"

    - set_fact:
        my_ssh_user: "{{ lookup('hashi_vault2', 'secret='+vault_secret_path+' token='+token+' url='+vault_url+' proxies='+proxies_str, errors='ignore').data.ssh_user }}"
        my_ssh_passphrase: "{{ lookup('hashi_vault2', 'secret='+vault_secret_path+' token='+token+' url='+vault_url+' proxies='+proxies_str, errors='ignore').data.ssh_passphrase | b64decode }}"
        my_ssh_private_key_data: "{{ lookup('hashi_vault2', 'secret='+vault_secret_path+' token='+token+' url='+vault_url+' proxies='+proxies_str, errors='ignore').data.ssh_private_key | b64decode }}"

    - debug:
        msg:
          - "my_ssh_user: {{ my_ssh_user }}"
          - "my_ssh_passphrase: {{ my_ssh_passphrase }}"
          - "my_ssh_private_key_data: {{ my_ssh_private_key_data }}"
    - copy: content="{{ my_ssh_private_key_data }}" dest=/tmp/my_ssh_private_key_file
    - name: Change file ownership, group and permissions
      file:
        path: /tmp/my_ssh_private_key_file
        mode: '0600'
```

This time we unset the environment variable and instead set the proxies as parameter set by the playbook vars.

`unset VAULT_PROXY`\
`ansible-playbook -i "localhost," retrieve_credentials_playbook2.yaml`

Ansible does not allow us to use the private key file contents to ssh to host endpoints. See the [reference](https://github.com/ansible/ansible/issues/22382 "Allow ansible_ssh_private_key variable") for more details. So we have to create a temporary file and add it to the ssh-agent. **Do not leave the private key retrieved from vault on the file system** even when the bubblewrap will delete it when invoked from Tower. For example, the file "/tmp/my_ssh_private_key_file" created above should be deleted by your playbook as soon as you add it to the ssh-agent. Also, the source code in the github has the no_log line commented out in multiple places so that we can see what is happening in the logs. However, it is prudent that you set "no_log: true" to avoid exposing the passwords and keys.

**Retrieving credentials from vault using Ansible Tower job over the tunnel with multiple hops**

The "test_vault_template" uses the playbook [test_vault.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/test_vault.yaml "test_vault.yaml") that is the same as the retrieve_credentials_playbook2.yaml with an additional play at the beginning that [establishes the tunnel](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/test_vault.yaml#L2-L7 "establishes the tunnel ") using a jumphost credential type. The extra variables passed are for the token, vault_secret_path and the vault_url. The [socks port is retrieved](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/test_vault.yaml#L16 "socks port") from the jumphost credential "yellowzone_5jumphost_credential_passphrase". It was set to socks5h://127.0.0.1:1239. We will retrieve the credential for target_hostname testaws-001.

![](images/Screen-Shot-2020-08-06-at-9.21.10-AM.png)

The output screenshot shows the playbook executed successfully. It retrieves credentials from the vault (credentials not shown):

![](images/Screen-Shot-2020-08-06-at-9.20.47-AM.png)

#### 6. Retrieving endpoint target inventory_hostnames

We want to retrieve the target inventory_hostnames that are selected when the job is called on the inventory with the LIMIT. This is required because we will use these names to retrieve credentials from vault. This is shown with the inventory_hostnames_template that invokes the [inventory_hostnames.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/inventory_hostnames.yaml "inventory_hostnames.yaml") playbook. The "yellowzone" inventory contains numerous aakubuntu*, aaksles*, aakrhel00* hosts and a group "amazon" containing testaws-001, testaws-002, testaws-003.

![](images/Screen-Shot-2020-08-06-at-9.33.25-AM.png)

We use the [following play](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/inventory_hostnames.yaml#L14-L20 "Collect applicable inventory_hostname") to collect the inventory_hostname of all applicable hosts that are targeted by the LIMIT into a temporary file with delegate_to localhost.

``` yaml
- name: Collect applicable inventory_hosts with delegate_to
  hosts: all
  gather_facts: false
  serial: 1
  tasks:
    - lineinfile: dest="{{ hostvars['127.0.0.1']['tempfile'] }}" line="{{ inventory_hostname }}" state=present create=yes
      delegate_to: localhost
```

 We use the applicable hosts collected in [inventory_hosts fact](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/inventory_hostnames.yaml#L30-L32 "inventory_hosts") below:

``` yaml
- name: Get the inventory_hosts
  shell: |
    cat {{ tempfile }}
  register: inventory_hosts_output
- name: Register the inventory_hosts
  set_fact:
    inventory_hosts: "{{ inventory_hosts_output.stdout_lines }}"
```

The output shows that only the hostnames selected with the LIMIT="aakrhel* amazon localhost" are added to the inventory_hosts list.

![](images/Screen-Shot-2020-08-06-at-9.33.03-AM.png)

Alternatively, we can use the magic variable ansible_play_hosts_all for "hosts: all" with "delegate_facts: true". The all_inventory_hosts can be used on the localhost in the next play.

``` yaml
- name: Collect applicable inventory_hosts with delegate_to method 1
  hosts: all
  gather_facts: false
  tasks:
    - set_fact:
        all_inventory_hosts: "{{ ansible_play_hosts_all }}"
      delegate_to: localhost
      delegate_facts: true
      run_once: true

- name: Show the all_inventory_hosts
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Show the targeted hosts
      debug:
        msg: "{{ all_inventory_hosts }}"
```

#### 7. Putting it all together in Ansible Tower

We will be using the custom [jumphost5_credential_type](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/Credentials.md "jumphost5_credential_type") to create a new credential. We can use either the unix socket file (for example "socks5h://unixsocket/tmp/mysocks") or the socket port (for example "socks5h://127.0.0.1:1234"). This custom credential will be used to create a tunnel to reach the vault over multiple hops. We need to retrieve the credentials for the endpoints from the vault. Although we should normally have a single credential targeting a group of host endpoints, shown below is a contrived example that shows how to retrieve separate ssh keys with corresponding passphrase for each target host from the vault and add all these ssh keys to the ssh-agent. The inventory may have a lot of hosts and we only want to target a subset of host endpoints as specified in the LIMIT. The first task is to retrieve this subset of hosts as shown in Section 6. Next, we create a target_hosts group and set the connection parameters by retrieving the credentials (user, ssh private key and passphrase) from the vault. We add the ssh keys to the ssh agent. Finally, we run the play on the target_hosts group such that it gets the ssh keys from the ssh-agent.

The credential lookup in the vault will only be done for the inventory_hosts as shown in the with_items loop below by including the [use_vault_include.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/use_vault_include.yaml "use_vault_include.yaml"):

``` yaml
vars:
  proxies: {"http":"{{ jh1_socks_port if jh1_socks_port is defined else jh_socks_port }}","https:":"{{ jh1_socks_port if jh1_socks_port is defined else jh_socks_port }}"}
tasks:
  - set_fact:
      proxies_str: "{{ proxies | to_json | replace(' ','') | string }}"
  - include_tasks: use_vault_include.yaml # Uses: proxies_str myhostname vault_secret_path token vault_url
    with_items: "{{ inventory_hosts }}"
    loop_control:
      loop_var: myhostname
```

The use_vault_include.yaml runs for each of the loop_var "myhostname", retrieves the credentials from the vault and runs the following to add the ssh keys to the ssh-agent:

``` yaml
- name: Set endpoint ssh private key for adding to ssh-agent in ansible-role-ssh-add-jumphosts-endpoint role
  set_fact:
    endpoint_ssh_private_key: "/tmp/my_ssh_private_key_file"
    endpoint_ssh_private_key_passphrase: "{{ my_ssh_passphrase }}"
    endpoint_ssh_user: "{{ my_ssh_user }}"
- name: add ssh key
  include_role:
    name: ansible-role-ssh-add-jumphosts-endpoint
```

You should be able to write a simple expect script to do this for adding the single key, but for this article it was easy to just clone the [previous role](https://github.com/thinkahead/DeveloperRecipes/tree/master/Jumphosts/roles/ansible-role-ssh-add-jumphosts-all "ansible-role-ssh-add-jumphosts-all") and modify [ansible-role-ssh-add-jumphosts-endpoint](https://github.com/thinkahead/DeveloperRecipes/tree/master/Jumphosts/roles/ansible-role-ssh-add-jumphosts-endpoint "ansible-role-ssh-add-jumphosts-endpoint") for use on a single endpoint.

If you pass a machine credential to the job template, then Tower starts the ansible-playbook process with "ssh-agent sh -c ssh-add" as follows:

`awx 4 3 0 12:52 ? 00:00:00 ssh-agent sh -c ssh-add /tmp/awx_13009_th27lw3y/artifacts/13009/ssh_key_data && rm -f /tmp/awx_13009_th27lw3y/artifacts/13009/ssh_key_data && ansible-playbook -u ec2-user -l 'testaws-001 testaws-003 localhost' -vvvvv -e @/tmp/awx_13009_th27lw3y/tmpnm8p0hws -i /tmp/awx_13009_th27lw3y/tmp6ri2lvw8 -e @/tmp/awx_13009_th27lw3y/env/extravars Jumphosts/use_vault.yaml`

In this scenario, we do not require/use the Machine credential to connect to hosts (it is ignored if passed) because we want to use the credentials from Vault. We can however reuse this ssh-agent to add target host endpoint ssh keys. If the Machine credential is not passed to the job, then this ssh-agent process is not present, and we have to start an ssh-agent. If we start a new ssh-agent, then we need to retrieve the environment variables "SSH_AUTH_SOCK" and "SSH_AGENT_PID" and set the -oIdentityAgent when establishing the ssh connection to the target host endpoints. If we reuse the ssh-agent started by Tower job, we do not need to set the "IdentityAgent" because it will use the correct environment variables. We set it anyway.

We will run the scenario in two ways:

1.  [use_vault.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/use_vault.yaml "use_vault.yaml") -- hosts: all -- This all targets all the hosts selected in the LIMIT or all hosts in the inventory if LIMIT is empty. The localhost must also be added to the LIMIT if you select specific hosts. The play with "hosts: all" does not run on localhost.
2.  [use_vault_with_group.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/use_vault_with_group.yaml "use_vault_with_group.yaml") -- hosts: target_hosts. A new group is created called target_hosts.

For the first case with "hosts: all", we use the job template "use_vault_template". It uses the playbook [use_vault.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/use_vault.yaml "use_vault.yaml"). We have the following hosts in the inventory "yellowzone" that we will use: aakrhel001, aakrhel002, testaws-001, testaws-002, testaws-003. We are passing the "yellowzone_5jumphost_credential_passphrase" custom credential with 5 jumphost hops with each hop having a ssh key with a passphrase. We will use this jumphost credential for two purposes:

1.  Retrieve host endpoint credentials from Vault over the jumphost hops
2.  Connect to endpoints over the jumphost hops

Connecting to the vault requires that we create a tunnel across the jumphosts and retrieve the endpoint credentials from the vault_secret_path as shown in the Section 5 using the custom lookup [hashi_vault2 plugin](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/lookup_plugins/hashi_vault2.py "hashi_vault2 plugin").

The first two target host endpoints: aakrhel001 and the aakrhel002 require that we connect via jumphosts. We will use different number of jumphost hops for each of these hosts. Therefore, we need the ansible_ssh_common_args set as host variables. The following shows the value for single jumphost hop set for aakrhel002.

`ansible_ssh_common_args: '-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ProxyCommand="ssh -W %h:%p {{ jh1_ssh_user if jh1_ssh_user is defined else jh_ssh_user }}@{{ jh1_ip if jh1_ip is defined else jh_ip }} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"'`

The following ansible_ssh_common_args is set to use 5 hops for connecting to aakrhel001. This mechanism to use alternating single and double quotes instead of backslashes to escape the double quotes was shown in last [Part 8](..//multiple-jumphosts-in-ansible-tower-part-8/ "Multiple Jumphosts in Ansible Tower - Part 8") when we used rsync in the file [synchronize.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/synchronize.yaml "synchronize.yaml")

Either of these escape mechanisms will work for nested ProxyCommand for ssh.

`ansible_ssh_common_args: -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='ssh -q -W %h:%p -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='"'"'ssh -q -W {{ jh5_ip }}:{{ jh5_ssh_port }} -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='"'"'"'"'"'"'"'"'ssh -q -W {{ jh4_ip }}:{{ jh4_ssh_port }} -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'ssh -q -W {{ jh3_ip }}:{{ jh3_ssh_port }} -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'ssh -q -W {{ jh2_ip }}:{{ jh2_ssh_port }} -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null {{ jh1_ssh_user }}@{{ jh1_ip }}'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"' {{ jh2_ssh_user }}@{{ jh2_ip }}'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"'"' {{ jh3_ssh_user }}@{{ jh3_ip }}'"'"'"'"'"'"'"'"' {{ jh4_ssh_user }}@{{ jh4_ip }}'"'"' {{ jh5_ssh_user }}@{{ jh5_ip }}'`

The aakaws-001, aakaws-002, aakaws-003 belong to a group called "amazon" in the same yellowzone inventory. These have direct connectivity from Tower and do not require a jumphost.

![](images/Screen-Shot-2020-08-05-at-2.31.02-PM.png)

The following inventory variables have been set to provide default user to login as and the default ssh port. Tower does not have Machine Credentials (because we only pass the Jumphost Credentials) to get the ansible_user, so ansible_user must be provided as inventory/group/host variable.

`ansible_user: ec2-user`\
`ansible_port: 22`

The job run executes successfully as shown in following screenshot:

![](images/Screen-Shot-2020-08-05-at-3.22.48-PM.png)

The output shows the ansible_ssh_common_args for the testaws-001 (same for testaws-002 and testaws-003) that only contains the IdentityAgent for the ssh-agent with the target endpoint's ssh key:

`"ansible_ssh_common_args='-oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 '",`

The output shows the ansible_ssh_common_args for the aakrhel002 that contains the ProxyCommand to use one hop as set in the host variables.

`"ansible_ssh_common_args": "-oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ProxyCommand=\"ssh -oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 -W %h:%p ec2-user@ec2-52-201-237-93.compute-1.amazonaws.com -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null\"",`

The output shows the ansible_ssh_common_args for the aakrhel001 that contains the ProxyCommand to use 5 hops as set in the host variables.

`"ansible_ssh_common_args": "-oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='ssh -oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 -q -W %h:%p -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='\"'\"'ssh -oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 -q -W aakrhel006.yellowykt.com:22 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='\"'\"'\"'\"'\"'\"'\"'\"'ssh -oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 -q -W aakrhel003.yellowykt.com:22 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'ssh -oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 -q -W aakrhel002.yellowykt.com:22 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'ssh -oIdentityAgent=/tmp/ssh-NYvQKRVOQTv3/agent.257 -q -W aakrhel001.yellowykt.com:22 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null ec2-user@ec2-52-201-237-93.compute-1.amazonaws.com'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"' ec2-user@aakrhel001.yellowykt.com'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"' ec2-user@aakrhel002.yellowykt.com'\"'\"'\"'\"'\"'\"'\"'\"' ec2-user@aakrhel003.yellowykt.com'\"'\"' ec2-user@aakrhel006.yellowykt.com'",`

For the second case with "hosts: target_hosts", we use the job template use_vault_with_group_template. It uses the playbook [use_vault_with_group.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/use_vault_with_group.yaml "use_vault_with_group.yaml"). We still get the credentials from the vault and add to the ssh-agent, however we do not need to set the ansible_port, ansible_user parameters in the Ansible Tower inventory/group/hosts variables. Instead these should also be set from the vault. Thus, the main difference from above first case is that in this new playbook, the included [use_vault_with_group_include.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/use_vault_with_group_include.yaml "use_vault_with_group_include.yaml") creates a group called target_hosts. This generated group contains the hosts along with the host variables that we want to use. We currently retrieve the ansible_user from the vault and set it here. Although not shown, we could retrieve the rest of the ansible connection parameters from vault and set them.  The ansible_ssh_private_key_file does not need to be set (commented below) because the ssh-add adds this file and deletes the private key.

``` yaml
- name: Adding to custom group
  add_host:
    name:  "{{ myhostname }}"
    groups: target_hosts
    #ansible_ssh_private_key_file: "{{ endpoint_ssh_private_key }}"
    ansible_ssh_user: "{{ endpoint_ssh_user }}"
    ansible_port: 22
    ansible_ssh_common_args2: "{{ '-oIdentityAgent='+hostvars['127.0.0.1']['env_vars']['SSH_AUTH_SOCK'] }}"
    #ansible_become_pass: "{{ ansible_become_password }}"
    #ansible_become_user: "{{ ansible_become_user }}"
    #ansible_become_method: su
    #ansible_become: True
```

This is shown in the execution of the use_vault_with_group_template that uses the playbook [use_vault_with_group.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/use_vault_with_group.yaml "use_vault_with_group.yaml")

![](images/Screen-Shot-2020-08-05-at-6.22.13-PM.png)

The output in screenshot below shown that it executed successfully on the 4 selected endpoints.

![](images/Screen-Shot-2020-08-05-at-6.26.12-PM.png)

The output msg for aakrhel001 is shown below. The ansible_user is retrieved from the vault, but the other values are hardcoded. We can have ansible_port and along with the [become connection variables](https://docs.ansible.com/ansible/latest/user_guide/become.html#become-connection-variables "become connection variables") also retrieved from vault. Additionally, we can retrieve the ansible_ssh_common_args_vault and update it to add the IdentityAgent and use this value. We currently set the ansible_ssh_common_args_vault='-oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237′ but we do not use it. Instead, we currently retrieve the ansible_ssh_common_args from the host variable as in the first case because we don't have these stored in the vault.

`    "msg": [\
        "ansible_ssh_common_args='-oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='ssh -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -q -W %h:%p -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='\"'\"'ssh -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -q -W aakrhel006.yellowykt.com:22 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='\"'\"'\"'\"'\"'\"'\"'\"'ssh -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -q -W aakrhel003.yellowykt.com:22 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'ssh -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -q -W aakrhel002.yellowykt.com:22 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand='\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'ssh -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -q -W aakrhel001.yellowykt.com:22 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null ec2-user@ec2-52-201-237-93.compute-1.amazonaws.com'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"' ec2-user@aakrhel001.yellowykt.com'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"' ec2-user@aakrhel002.yellowykt.com'\"'\"'\"'\"'\"'\"'\"'\"' ec2-user@aakrhel003.yellowykt.com'\"'\"' ec2-user@aakrhel006.yellowykt.com''",\
        "ansible_ssh_common_args_vault='-oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237'",\
        "**ansible_ssh_common_args_role**='-oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand=\"ssh -i /tmp/awx_13077_wij9x19o/tmph6xsnyuc -W %h:%p -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand=\\\"ssh -i /tmp/awx_13077_wij9x19o/tmpjpssz5w4 -W aakrhel006.yellowykt.com:22 -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand=\\\\\\\"ssh -i /tmp/awx_13077_wij9x19o/tmpwrqcrhpy -W aakrhel003.yellowykt.com:22 -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand=\\\\\\\\\\\\\\\"ssh -i /tmp/awx_13077_wij9x19o/tmps70_ytzx -W aakrhel002.yellowykt.com:22 -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oProxyCommand=\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\"ssh -i /tmp/awx_13077_wij9x19o/tmprgs4ul1g -W aakrhel001.yellowykt.com:22 -oIdentityAgent=/tmp/ssh-WzBQZqacxWiP/agent.237 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null ec2-user@ec2-52-201-237-93.compute-1.amazonaws.com\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\" ec2-user@aakrhel001.yellowykt.com\\\\\\\\\\\\\\\" ec2-user@aakrhel002.yellowykt.com\\\\\\\" ec2-user@aakrhel003.yellowykt.com\\\" ec2-user@aakrhel006.yellowykt.com\"'",\
        "ansible_port=22",\
        "ansible_user=ec2-user"\
    ],`

Shown above is the **ansible_ssh_common_args_role** generated from the role ansible-role-ssh-add-jumphosts-all. We use it in previous parts. In this part, we do not use this value because we want to use a custom ansible_ssh_common_args. We could have used it if all jumphosts required the same number of hops as defined by the jumphost credential.

In the next Section 8, we switch to the CyberArk Conjur.

#### 8. Example setup with the CyberArk Conjur

We will start by installing a test instance of Conjur OpenSource using Docker on a VM (aakrhel001.yellowykt.com) accessible from the jumphost ec2-52-201-237-93.compute-1.amazonaws.com

On aakrhel001.yellowykt.com, we run the following:

`git clone https://github.com/cyberark/conjur-quickstart.git`

`Cloning into 'conjur-quickstart'...`\
`remote: Enumerating objects: 4, done.`\
`remote: Counting objects: 100% (4/4), done.`\
`remote: Compressing objects: 100% (4/4), done.`\
`remote: Total 174 (delta 0), reused 2 (delta 0), pack-reused 170`\
`Receiving objects: 100% (174/174), 46.23 KiB | 0 bytes/s, done.`\
`Resolving deltas: 100% (60/60), done.`

`cd conjur-quickstart/`

`docker-compose pull`

`Pulling openssl  ... done`\
`Pulling bot_app  ... done`\
`Pulling database ... done`\
`Pulling conjur   ... done`\
`Pulling proxy    ... done`\
`Pulling client   ... done`

`docker-compose run --no-deps --rm conjur data-key generate > data_key`

`Creating network "conjur-quickstart_default" with the default driver`

`export CONJUR_DATA_KEY="$(< data_key)"`

`docker-compose up -d`

`Creating postgres_database ... done`\
`Creating bot_app           ... done`\
`Creating openssl           ... done`\
`Creating conjur_server     ... done`\
`Creating nginx_proxy       ... done`\
`Creating conjur_client     ... done`

`docker-compose exec conjur conjurctl account create myConjurAccount > admin_data`

`docker-compose exec client conjur init -u conjur -a myConjurAccount`

`Wrote configuration to /root/.conjurrc`

`# This shows the api_key that we will use to retrieve the token and as admin's password`

`cat admin_data`

`Created new account 'myConjurAccount'`\
`Token-Signing Public Key: -----BEGIN PUBLIC KEY-----`\
`<public key>`\
`-----END PUBLIC KEY-----`\
`API key for admin: 2bdy4fwzn8tab4atdtm2cf32p68hsr9934946pc2wqj4kca34v5p`

`docker-compose exec client conjur authn login -u admin`

`Please enter admin's password (it will not be echoed):`\
`Logged in`

`docker-compose exec client conjur policy load root policy/BotApp.yml > my_app_data`

`docker-compose exec client conjur authn logout`

`Logged out`

`# This contains the api_keys for the use Dave and host BotApp`

`cat my_app_data`

`Loaded policy 'root'`\
`{`\
`  "created_roles": {`\
`    "myConjurAccount:user:Dave@BotApp": {`\
`      "id": "myConjurAccount:user:Dave@BotApp",`\
`      "api_key": "1m5qyc427naqtr2eb2mxbbr9y9t08cg531btxc216t043shqcjm3"`\
`    },`\
`    "myConjurAccount:host:BotApp/myDemoApp": {`\
`      "id": "myConjurAccount:host:BotApp/myDemoApp",`\
`      "api_key": "1afvhpd2dg1b01w9c77h2vvdxaj1zehzsg3fbe0vm1a226pn2vx3w4w"`\
`    }`\
`  },`\
`  "version": 1`\
`}`

`docker-compose exec client conjur authn login -u Dave@BotApp`

`Please enter Dave@BotApp's password (it will not be echoed):`\
`Logged in`

`docker-compose exec client conjur authn whoami`

`{"account":"myConjurAccount","username":"Dave@BotApp"}`

`# Show the sample policy file, we will update it below`

`docker-compose exec client cat policy/BotApp.yml`

`docker-compose exec client bash`

`vi policy/BotApp.yml # Create the file as shown below`

``` yaml
- !policy
  id: BotApp
  body:
    # Define a human user, a non-human identity that represents an application, and a secret
  - !user Dave
  - !host myDemoApp
  - &ssh_credentials
    - !variable testsecrets-jumphost-ssh_user
    - !variable testsecrets-jumphost-ssh_private_key
    - !variable testsecrets-jumphost-ssh_passphrase
    - !variable testsecrets-aakrhel001-ssh_user
    - !variable testsecrets-aakrhel001-ssh_private_key
    - !variable testsecrets-aakrhel001-ssh_passphrase
    - !variable testsecrets-aakrhel002-ssh_user
    - !variable testsecrets-aakrhel002-ssh_private_key
    - !variable testsecrets-aakrhel002-ssh_passphrase
    - !variable testsecrets-testaws-001-ssh_user
    - !variable testsecrets-testaws-001-ssh_private_key
    - !variable testsecrets-testaws-001-ssh_passphrase
    - !variable testsecrets-testaws-002-ssh_user
    - !variable testsecrets-testaws-002-ssh_private_key
    - !variable testsecrets-testaws-002-ssh_passphrase
    - !variable testsecrets-testaws-003-ssh_user
    - !variable testsecrets-testaws-003-ssh_private_key
    - !variable testsecrets-testaws-003-ssh_passphrase
  - !permit
    # Give permissions to the human user to update the secret and fetch the secret.
    role: !user Dave
    privileges: [read, update, execute]
    resource: *ssh_credentials
  - !permit
    # Give permissions to the non-human identity to fetch the secret.
    role: !host myDemoApp
    privileges: [read, execute]
    resource: *ssh_credentials
```

`conjur policy load --replace root policy/BotApp.yml`

Now we need to add our machine credentials to Conjur. We want to do that using the REST API. To find out what the curl commands should look like. Here is the [documentation](https://docs.conjur.org/latest/en/Content/Tools/CLI_Help.htm "Finding the curl command") on how to find that:

    Before you run a CLI command, use RESTCLIENT_LOG=stderr conjur <command> to see a list of the API queries used by the CLI.
    RestClient is a gem Conjur uses in the CLI to make REST API calls and it supports debug mode with the RESTCLIENT_LOG environment variable.
    For example, to see the list of API queries used by authn login:
    $ RESTCLIENT_LOG=stderr conjur authn login
    This syntax sets the environment variable RESTCLIENT_LOG to the value of stderr for the specified command.
    You can redirect the output to a file:
    $ export RESTCLIENT_LOG=conjur.log

While exec-ed into the client container, the following snippet shows the two RestClient.post required to add the value ec2-user to BotApp/testsecrets-jumphost-ssh_user. We will use this information for creating curl requests later.

`root@22fecf1a74b4:/# export RESTCLIENT_LOG=conjur.log`\
`root@22fecf1a74b4:/# rm -f conjur.log`\
`root@22fecf1a74b4:/# conjur variable values add BotApp/testsecrets-jumphost-ssh_user ec2-user`\
`Value added`\
`root@22fecf1a74b4:/# cat conjur.log`\
`RestClient.post "http://conjur/authn/myConjurAccount/admin/authenticate", "2bdy4fwzn8tab4atdtm2cf32p68hsr9934946pc2wqj4kca34v5p", "Accept"=>"*/*", "Content-Length"=>"52", "Content-Type"=>"text/plain", "User-Agent"=>"rest-client/2.1.0 (linux-gnu x86_64) ruby/2.5.1p57"`\
`# => 200 OK | application/json 612 bytes, 0.02s`\
`RestClient.post "http://conjur/secrets/myConjurAccount/variable/BotApp%2Ftestsecrets-jumphost-ssh_user", "ec2-user", "Accept"=>"*/*", "Authorization"=>"Token token=\"eyJwcm90ZWN0ZWQiOiJleUpoYkdjaU9pSmpiMjVxZFhJdWIzSm5MM05zYjNOcGJHOHZkaklpTENKcmFXUWlPaUk1WkRVNVpXWmxOR1V4TlRKaE1USTJOekl4T0RGa01tWXdOMlJoTmpNelpUTmhPRGMzTVdWbU5XVXhOV0ZpTVRVM056aGhPRFUyTjJOa1ptWXdaalU0SW4wPSIsInBheWxvYWQiOiJleUp6ZFdJaU9pSmhaRzFwYmlJc0ltbGhkQ0k2TVRVNU5qa3lNVEl6Tm4wPSIsInNpZ25hdHVyZSI6Imo0VUVwX1RiY0w3QWVNazRZYVhOeDZXYjczNFozV284R2cwdzBpNnFjMFdTOTBCQ3R1S1hVM3hlR1lrSGxWRFdSaXA0c19zYWNwM3VFRVhtdzBUVlNaSEtTQS12ZHJUdU53UmNBYUlBTjlqWTh5bklUMnZOdUxVN2ZScU1acWZ2VVFBdUZINzV1eVZuNVpidEYzYzc0NEMtUkw5U2xISWV2YklidTNUOElLVnlXckV2SHFLVV80ZXdKbWl6Y04yRVJnS0RLdUNuR3dxQWd4RWVTdXNvVXRDYjFlblc1eWNGZ2FsaWdLb3J0cnN0QTJCM0dLcVFwMGZFTUNHMUh2bkg3UUlKZjBKUk9iOHNyc0dBekhRUDFxU3FjRWhNTUhscy15bERiYWhwbHBqTm5HLXRjX2F1dmtCSU5SdURzTU5ZU1M2OVJNQlBWMjdNOWxpbWZzWjVzbnFxR1E1c1ZidUd4MXVDbGJkcGxMTlpOUWQ3MHVlVURCb2ZRcGRNZGRMeiJ9\"", "Content-Length"=>"8", "User-Agent"=>"rest-client/2.1.0 (linux-gnu x86_64) ruby/2.5.1p57"`\
`# => 201 Created | text/html 0 bytes, 0.03s`

We setup a tunnel from our Laptop to the jumphost ec2-52-201-237-93.compute-1.amazonaws.com that has access to the Conjur running in docker on aakrhel001.yellowykt.com:

`ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -CfNq  -D 127.0.0.1:1234 -p 22 -i ~/amazontestkey.pem ec2-user@ec2-52-201-237-93.compute-1.amazonaws.com -v`

Most API calls require an authentication access token in the header. Here's how to obtain it:

1\. Use a username and password to obtain an API key (refresh token) with the [Authentication > Login](https://docs.conjur.org/Latest/en/Content/Developer/Conjur_API_Login.htm "Conjur_API_Login") method.

2\. Use the API key to obtain an access token with the [Authentication > Authenticate](https://docs.conjur.org/Latest/en/Content/Developer/Conjur_API_Authenticate.htm "Conjur_API_Authenticate") method.

Access tokens expire after 8 minutes. You need to obtain a new token after it expires.

Now we can use the above information to run our requests remotely. We can alternatively retrieve the token using the REST API curl commands as follows:

1\. **Read/Execute access** -- host/BotApp/myDemoApp

`# docker-compose exec client conjur authn login -u host/BotApp/myDemoApp\
curl -d "1afvhpd2dg1b01w9c77h2vvdxaj1zehzsg3fbe0vm1a226pn2vx3w4w" --proxy socks5h://localhost:1234 -k https://aakrhel001.yellowykt.com:8443/authn/myConjurAccount/host%2FBotApp%2FmyDemoApp/authenticate > /tmp/conjur_token\
CONT_SESSION_TOKEN=$(cat /tmp/conjur_token| base64 | tr -d '\r\n')`

2\. **Write/Read/Execute access** -- Dave@BotApp

`# docker-compose exec client conjur authn login -u Dave@BotApp\
curl -d "1m5qyc427naqtr2eb2mxbbr9y9t08cg531btxc216t043shqcjm3" --proxy socks5h://localhost:1234 -k https://aakrhel001.yellowykt.com:8443/authn/myConjurAccount/Dave%40BotApp/authenticate > /tmp/conjur_write_token\
CONT_SESSION_TOKEN=$(cat /tmp/conjur_write_token| base64 | tr -d '\r\n')\
echo $CONT_SESSION_TOKEN`

3\. **Admin access** -- admin

`# docker-compose exec client conjur authn login -u admin\
curl -d "2bdy4fwzn8tab4atdtm2cf32p68hsr9934946pc2wqj4kca34v5p"  --proxy socks5h://localhost:1234 -k https://aakrhel001.yellowykt.com:8443/authn/myConjurAccount/admin/authenticate > /tmp/conjur_admin_token\
CONT_SESSION_TOKEN=$(cat /tmp/conjur_admin_token| base64 | tr -d '\r\n')\
echo $CONT_SESSION_TOKEN`

Once we have the token with required access, we can add the credentials. The ssh_user, ssh_private_key and ssh_passphrase are added below for the hosts defined in the BotApp policy. You can update the policy if you need to add additional hosts.

`for i in {1..3}; do`\
&nbsp;&nbsp;`  curl -X POST --proxy socks5h://localhost:1234 -s -k -H "Content-Type: text/plain" -H "Authorization: Token token=\"$CONT_SESSION_TOKEN\"" https://aakrhel001.yellowykt.com:8443/secrets/myConjurAccount/variable/BotApp%2Ftestsecrets-testaws-00$i-ssh_user -d ec2-user`\
&nbsp;&nbsp;``  curl -X POST --proxy socks5h://localhost:1234 -s -k -H "Content-Type: text/plain" -H "Authorization: Token token=\"$CONT_SESSION_TOKEN\"" https://aakrhel001.yellowykt.com:8443/secrets/myConjurAccount/variable/BotApp%2Ftestsecrets-testaws-00$i-ssh_private_key -d `cat ~/Downloads/expecttest/alexei-key-ecdsa-testaws-00$i | base64` ``\
&nbsp;&nbsp;``  curl -X POST --proxy socks5h://localhost:1234 -s -k -H "Content-Type: text/plain" -H "Authorization: Token token=\"$CONT_SESSION_TOKEN\"" https://aakrhel001.yellowykt.com:8443/secrets/myConjurAccount/variable/BotApp%2Ftestsecrets-testaws-00$i-ssh_passphrase -d `echo Passphrase4testaws-00$i | base64` ``\
`done`

Retrieve the ssh_user for testaws-003

`curl --proxy socks5h://localhost:1234 -s -k -H "Content-Type: text/plain" -H "Authorization: Token token=\"$CONT_SESSION_TOKEN\"" https://aakrhel001.yellowykt.com:8443/secrets/myConjurAccount/variable/BotApp%2Ftestsecrets-testaws-003-ssh_user -vvv`

It returns ec2-user. Now we can run the following [test_conjur.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/test_conjur.yaml "test_conjur.yaml") playbook to retrieve the credentials for target_hosts.

``` yaml
- hosts: localhost
  gather_facts: no
  connection: local
  vars:
    api_key: ""
    app_id: BotApp
    auth_secret_path: "/authn/myConjurAccount/admin/authenticate"
    vault_secret_path: "/secrets/myConjurAccount/variable/{{ app_id }}%2F"
    vault_url: "https://aakrhel001.yellowykt.com:8443"
    proxy: "{{ jh1_socks_port if jh1_socks_port is defined else jh_socks_port }}"
    target_hosts: ["aakrhel001","aakrhel002","testaws-001","testaws-002","testaws-003"]
  tasks:
    - debug:
        msg:
          - "vault_url: {{ vault_url }}"
          - "vault_secret_path: {{ vault_secret_path }}"

    - name: Retrieve the token and use it to retrieve the credentials for all target_hosts
      shell: curl -s -d "{{ api_key }}" --proxy socks5h://localhost:1234 -k {{ vault_url }}{{ auth_secret_path }} | base64 | tr -d '\r\n'
      register: token_output
    - fail:
        msg: Count not retrieve token from {{ auth_secret_path }}
      when: token_output.stdout|length == 0
    - set_fact: token="{{ token_output.stdout }}"
    - name: Retrieve the credentials
      shell: |
        my_ssh_user=`curl --proxy {{ proxy }} -s -k -H "Content-Type: text/plain" -H "Authorization: Token token=\"{{ token }}\"" {{ vault_url }}{{ vault_secret_path }}testsecrets-{{ target_host }}-ssh_user 2>/dev/null`
        my_ssh_passphrase=`curl --proxy {{ proxy }} -s -k -H "Content-Type: text/plain" -H "Authorization: Token token=\"{{ token }}\"" {{ vault_url }}{{ vault_secret_path }}testsecrets-{{ target_host }}-ssh_passphrase 2>/dev/null | base64 -d`
        my_ssh_private_key=`curl --proxy {{ proxy }} -s -k -H "Content-Type: text/plain" -H "Authorization: Token token=\"{{ token }}\"" {{ vault_url }}{{ vault_secret_path }}testsecrets-{{ target_host }}-ssh_private_key 2>/dev/null | base64 -d`
        echo '{ "{{ target_host }}": {"ssh_user":"'$my_ssh_user'","ssh_passphrase":"'$my_ssh_passphrase'","ssh_private_key":"'$my_ssh_private_key'"}}'
      with_items: "{{ target_hosts }}"
      loop_control:
        loop_var: target_host
      register: response
    - debug:
        msg: "{{ response.results }}"
    - set_fact:
        creds: "{{ {} }}"
    - set_fact:
        creds: "{{ creds | combine(item|from_json) }}"
      with_items: "{{ response.results | json_query('[*].stdout[]') }}"
    - debug:
        msg: "{{ creds['aakrhel001'] }}"
```

This uses the admin url to authenticate and therefore requires the admin token. However if you use the Dave@BotApp user or the host/BotApp/myDemoApp, you can use the coresponding url from above for the auth_secret_path.

`ansible-playbook test_conjur.yaml -e jh_socks_port=socks5h://127.0.0.1:1234 -e api_key=2bdy4fwzn8tab4atdtm2cf32p68hsr9934946pc2wqj4kca34v5p -vvv`

We can create a job template "test_conjur_template" as shown in screenshot below in Ansible Tower and run the [test_conjur.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/test_conjur.yaml "test_conjur.yaml") playbook that also establishes a tunnel by [including the role](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/test_conjur.yaml#L2-L7 "include the role ansible-role-socks5-tunnel"). The screenshot also shows the three possibilities for the auth_secret_path.

![](images/Screen-Shot-2020-08-08-at-7.22.16-PM.png)

The curl commands used in test_conjur.yaml playbook above do not work with unix socket file. A sample python module to retrieve the token [library/get_conjur_token.py](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/library/get_conjur_token.py "Module get_conjur_token") using socks proxy that works with socks port and unix socket file is shown below:

``` python
import time
from ansible.module_utils.basic import AnsibleModule
import requests
import base64
import json

def main():
    fields = {
        "url": {"type": "str"},
        "data": {"type": "str"},
        "proxies": {"type": "dict"},
        "verify": {"type": "bool"}
    }
    module = AnsibleModule(argument_spec=fields)
    url  = module.params['url']
    data  = module.params['data']
    proxies  = module.params['proxies']
    verify  = module.params['verify']
    r=requests.post(url, proxies=proxies, data=data, verify=verify)
    module.exit_json(changed=True, content=r.content, status_code=r.status_code)

if __name__ == '__main__':
    main()
```

A sample module [library/get_conjur_variables.py](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/library/get_conjur_variables.py "Module get_conjur_variables") to retrieve conjur variables that should be run on localhost is shown below:

``` python
import time
from ansible.module_utils.basic import AnsibleModule
import requests
import base64
import json

def main():

    fields = {
        "url": {"type": "str"},
        "token": {"type": "str"},
        "attributes": {"type": "list"},
        "proxies": {"type": "dict"},
        "verify": {"type": "bool"}
    }
    module = AnsibleModule(argument_spec=fields)
    url  = module.params['url']
    token  = module.params['token']
    attributes  = module.params['attributes']
    proxies  = module.params['proxies']
    verify  = module.params['verify']
    result={}
    for attribute in attributes:
        result[attribute]=requests.get(url+attribute, proxies=proxies, verify=verify, headers={"Authorization": "Token token=\""+token+"\""}).content
    module.exit_json(changed=True, result=result)

if __name__ == '__main__':
    main()
```

The sample playbook [test_conjur2.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/test_conjur2.yaml "test_conjur2.yaml") shows how to use the above two modules to retrieve the credentials when running a job from Ansible Tower.

A lookup plugin [lookup_plugins/conjur_vault.py](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/lookup_plugins/conjur_vault.py "lookup_plugins/conjur_vault.py") to retrieve the credentials from Conjur is shown below:

``` python
from __future__ import (absolute_import, division, print_function)
__metaclass__ = type

DOCUMENTATION = """
  lookup: conjur_vault
  author: Alexei Karve <karve(at)us.ibm.com>
  version_added: "2.0"
  short_description: retrieve secrets from Conjur vault
  description:
    - retrieve secrets from Conjur vault
  options:
    token:
      description: vault token.
    target_hosts:
      description: target_hosts for which we want to retrieve attributes.
      default: None
    attributes:
      description: variables to retrieve.
      default: ['ssh_user','ssh_passphrase','ssh_private_key']
    proxies:
      description: Socks proxies.
      default: proxies={"http":"socks5h://localhost:1234","https":"socks5h://localhost:1234"}
    url:
      description: URL to vault service.
    verify:
      description: controls verification and validation of SSL certificates, only want to turn off with self signed ones.
      type: boolean
      default: True
"""

EXAMPLES = """
- debug:
    msg: "{{ lookup('conjur_vault','url=' ~ vault_url ~ vault_secret_path ~ 'testsecrets-aakrhel001 token=' ~ token ~ ' proxies=' ~ proxies_str ~ ' verify=False') }}"

"""

RETURN = """
_raw:
  description:
    - secrets(s) requested
"""

import os
import json
import requests

from ansible.errors import AnsibleError
from ansible.module_utils.parsing.convert_bool import boolean
from ansible.plugins.lookup import LookupBase

class LookupModule(LookupBase):
    def run(self, terms, variables=None, **kwargs):
        vault_args = terms[0].split()
        vault_dict = {}
        ret = []

        for param in vault_args:
            try:
                key, value = param.split('=',1)
            except ValueError:
                raise AnsibleError("conjur_vault lookup plugin needs key=value pairs, but received %s" % param)
            vault_dict[key] = value

        url = vault_dict['url']
        token = vault_dict['token']
        if vault_dict.get('target_hosts'): target_hosts = json.loads(vault_dict['target_hosts'])
        else: target_hosts=None
        if vault_dict.get('attributes'): attributes  = json.loads(vault_dict['attributes'])
        else: attributes = ['ssh_user','ssh_passphrase','ssh_private_key']
        if vault_dict.get('proxies'): proxies = json.loads(vault_dict['proxies'])
        else: proxies={"http":"socks5h://localhost:1234","https":"socks5h://localhost:1234"}
        verify=vault_dict.get('verify',True)
        if target_hosts is None or len(target_hosts)==0:
            result={}
            for attribute in attributes:
                result[attribute]=requests.get(url+"-"+attribute, proxies=proxies, verify=False, headers={"Authorization": "Token token=\""+token+"\""}).content
            ret.append(result)
        else:
            results={}
            for target_host in target_hosts:
                result={}
                for attribute in attributes:
                    result[attribute]=requests.get(url+"-"+target_host+"-"+attribute, proxies=proxies, verify=False, headers={"Authorization": "Token token=\""+token+"\""}).content
                results[target_host]=result
            ret.append(results)
        ret.append(url)
        ret.append(target_hosts)
        ret.append(attributes)
        return ret
```

The sample playbook [test_conjur3.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/Jumphosts/test_conjur3.yaml "test_conjur3.yaml") shows how to use the above lookup plugin to retrieve the credentials from a job in Ansible Tower.

This section showed how to retrieve credentials from Conjur over multiple jumphost hops.

#### 9. Conclusion

In Part 9, we saw how to access a vault over multiple jumphost hops to retrieve host endpoint credentials with curl, python code and lookup plugins from Ansible. We further added these retrieved ssh keys to the ssh-agent and connected to the target host endpoints. We can store other cloud credentials (for example, Amazon Web Services requires access keys and secret keys) or vCenter credentials in the vault. We can retrieve and use these credentials to run delegated playbooks or shell commands or web requests (REST API) on an endpoint or jumphost that has access to the APIs over multiple jumphost hops. In [Part 10](..//multiple-jumphosts-in-ansible-tower-part-10 "Multiple Jumphosts in Ansible Tower - Part 10"), we will use customized ansible-runner images with Container Groups.

Hope you have enjoyed the article. Share your thoughts in the comments or engage in the conversation with me on Twitter @aakarve. I look forward to hearing about which vaults and plugins you use to retrieve credentials using Ansible and if you would like to see something covered in more detail.

#### 10. References
- HVAC Python Client for Hashicorp Vault <https://readthedocs.org/projects/hvac/downloads/pdf/stable/>
- Retrieve secrets from HashiCorp's vault <https://raw.githubusercontent.com/ansible/ansible/stable-2.9/lib/ansible/plugins/lookup/hashi_vault.py>
- Setting the Environment in Ansible <https://docs.ansible.com/ansible/latest/user_guide/playbooks_environment.html>
- Adding a plugin locally <https://docs.ansible.com/ansible/latest/dev_guide/developing_locally.html#adding-a-plugin-locally>
- Allow ssh private keys to be used that are encrypted with vault <https://github.com/ansible/ansible/issues/22382>
- Add ansible_ssh_private_key for paramiko_ssh <https://github.com/ansible/ansible/pull/22764>
- Conjur Quick Start <https://www.conjur.org/get-started/quick-start/oss-environment/>
- Multiple Jumphosts in Ansible Tower -- Part 1: Connecting to Linux hosts using ssh with nested ProxyCommand <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-1>
- Multiple Jumphosts in Ansible Tower -- Part 2: Connecting to Windows/Linux hosts with ssh tunnel SOCKS5 proxy <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-2>
- Multiple Jumphosts in Ansible Tower -- Part 3: Ssh tunnel SOCKS5 proxy with passphrase enabled for ssh keys <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-3>
- Multiple Jumphosts in Ansible Tower -- Part 4: Multi jumphost connections to Linux hosts using ssh-add to add keys to ssh-agent <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-4/>
- Multiple Jumphosts in Ansible Tower -- Part 5: Unix domain socket file instead of socks port <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-5/>
- Multiple Jumphosts in Ansible Tower -- Part 6: Primary and Secondary/Backup Jumphosts and Reverse ssh Tunnel <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-6/>
- Multiple Jumphosts in Ansible Tower -- Part 7: Failover using redundant jumphosts, tweaking ssh parameters and memory requirements for jumphosts <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-7/>
- Multiple Jumphosts in Ansible Tower -- Part 8: Transferring files using ansible synchronize module and rsync <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-8/>
- Multiple Jumphosts in Ansible Tower -- Part 10: Using Container Groups instead of bubblewrap for isolation of jobs <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-10/>
- Multiple Jumphosts in Ansible Tower -- Part 11: Using Container Groups with sshuttle proxy <https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-11/>
