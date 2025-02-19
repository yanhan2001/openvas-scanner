# OpenVAS Daemon

Is the implementation for [scanner-api](https://greenbone.github.io/scanner-api/).

Currently it is utilizing ospd-openvas.

# Requirements

It requires a running ospd-openvas instance.

# Configuration

Create a configuration file under either:

- `/etc/openvasd/openvasd.toml`
- `$HOME/.config/openvasd/openvasd.toml`

or provide the `-c` flag when starting openvasd.

Configure it like:


```
[feed]
# path to the openvas feed. This is required for the /vts endpoint.
path = "/var/lib/openvas/plugins"

[feed.check_interval]
# how often the feed should be checked for updates
secs = 3600
nanos = 0

[endpoints]
# enables GET /scans endpoint
enable_get_scans = true
# if set it requires `x-api-key` header to use the endpoint
key = "mtls_is_preferred"

[tls]
# the server certificate
certs = "/etc/openvasd/tls/server.pem"
# server key
key = "/var/lib/openvasd/tls/server.rsa"
# dir that contains client certificates. if there are none than every client is
# allowed to connect otherwise just the clients that have the configured
# client certificates
client_certs = "/etc/openvasd/tls/client"

[ospd]
# path to the unix socket of ospd-openvas
socket = "/var/run/ospd/ospd.sock"

[ospd.result_check_interval]
# interval of checking for results for started scans
secs = 1
nanos = 0

[listener]
# ip address and port to listen to
address = "127.0.0.1:3000"

[log]
# level of the log messages: TRACE > DEBUG > INFO > WARN > ERROR
level = "INFO"
```
## Authentication

The API supports two kinds of authentication methods:
    
- API Key
- Certificates

The authentication modes are set within a configuration file or via the argument list, when starting the server.

The authentication is required for each request except for a HEAD request.

### API Key

An API key is a token that the client provides when doing API requests and are used to authorize access.

If you want to enforce the requirement for clients to provide an x-api-key
header, make sure to set the key configuration option under [endpoints] to the
desired value. Clients connecting to the service will need to include this
header with the configured value for successful authentication.

The `X-API-KEY` must be in the header

`curl --insecure --request GET https://localhost:3000/scans -H "X-API-KEY: mtls_is_preferred"`

### Certificates
    
Both methods TLS and Mutual TLS (mTLS) are supported for authentication, while the second is strongly recommended, since it guaranties that both network connection ends are who they say to be.

If you want to enable TLS for secure communication, ensure that the TLS
certificate files are in place and accessible.

Server Certificates: The server certificate and private key files should be
located at the paths specified in the tls.certs and tls.key configuration
options, respectively. Adjust the file paths in the configuration if necessary.

Client Certificates: If you want to restrict access to clients with specific
client certificates, ensure that the directory specified in the
tls.client_certs configuration option contains the required client certificate
files.

#### How does mTLS works?

It is similar to the known TLS, but there are some extra steps. In the following list of steps for stablishing a mTLS connexion, the steps written in bold are the extra steps for mTLS:
1. The client connects to the server
2. The server presents its TLS certificate
3. The client verifies the server certificate
4. **The client presents its TLS certificate**
5. **The server verifies the client certificate**
6. **The server gives access to the client**
7. Client and server exchange information through a encrypted TLS connexion.

#### Certificate Authority for PKI mTLS Method

Who implements mTLS is its own Certification Authority. This is a difference from standard TLS, where an external organization verifies and validates the TLS certificates.
For mTLS is necessary a TLS root certificate which allows an organization to be its owns certificate authority. The root CA signs intermediate certificates forming a chain of trust. The intermediate CA is created for security reasons. then, the root certificate is used as less as possible and in case the intermediate key is compromised, the root CA can revoke the intermediate key and create a new one.
Finally, the intermediate CA is used to sign client and server certificates.

In the examples folder for a [PKI Mutual TLS](../examples/tls/PKI mTLS Method) you can find three scripts for creation of root and intermediate CA, client and server pairs (keys and certs).

Once you created the all key-certs pairs, you can use them for stablishing a mTLS secure connexion between `openvasd` and the clients.

In the openvasd.toml configuration file, under the section `[tls]`:
- set the variable `certs` with the path to the server certificate,
- set the variable `key` with the path to the server key,
- set the variable `client_certs` with the path to the intermediate CA certificate.

You should store in a secure place the root and intermediate key, which are not necessary anymore, but only for revoke/create intermediate CA pairs.

On the client side, you use the client key, the client cert and the same intermediate CA certificate you use in the server side. An example of usage is the following curl command:

`curl --insecure --verbose  --cert client.cert --key client.key --cacert CA/ca.cert --request GET https://localhost:3000/scans -H "X-API-KEY: mtls_is_preferred"`

Now, since both server and client are signed by the same CA, certificates can be verified and the encrypted connexion is authenticated.

#### mTLS with self-signed client certificates.

This method is similar to the explained above, but differs in that the clients and server certificates are self-signed, or signed by different CAs.
Then, the clients have to register a single certificate with the authorization server, and there is no shared CA certificate.
During authentication, the server checks if the client uses the same certificate for the TLS session as was configured or registered for that individual client. The server trusts the pinned certificate. The client's organization does not have to maintain any public key infrastructure and can simply use a self-signed certificate for authentication.

The scripts for generating the self-signed certificate can be found in [Self-Signed mTLS Method](../examples/tls/Self-Signed mTLS Method) folder.

Once you created the all key-certs pairs, you can use them for stablishing a mTLS secure connexion between `openvasd` and the clients.

In the openvasd.toml configuration file, under the section `[tls]`:
- set the variable `certs` with the path to the server certificate,
- set the variable `key` with the path to the server key,
- set the variable `client_certs` with the path to the folder with all registered client's self-signed certificates, previously shared via a secure method.

On the client side, you use the client key and the client certificate. An example of usage is the following curl command:

`curl --insecure --verbose  --cert client.cert --key client.key --request GET https://localhost:3000/scans -H "X-API-KEY: mtls_is_preferred"`

As can be seen, no CA certificate is used, since instead the client certificate is used on the server side.


# Usage

```
Usage: openvasd [OPTIONS]

Options:
  -c, --config <config>
          path to toml config file [env: OPENVASD_CONFIG=]
      --feed-path <feed-path>
          path to openvas feed [env: FEEED_PATH=]
      --feed-check-interval <SECONDS>
          interval to check for feed updates in seconds [env: FEED_CHECK_INTERVAL=]
      --tls-certs <tls-certs>
          path to server tls certs [env: TLS_CERTS=]
      --tls-key <tls-key>
          path to server tls key [env: TLS_KEY=]
      --tls-client-certs <tls-client-certs>
          path to client tls certs. Enables mtls. [env: TLS_CLIENT_CERTS=]
      --enable-get-scans
          enable get scans endpoint [env: ENABLE_GET_SCANS=]
      --api-key <api-key>
          API key that must be set as X-API-KEY header to gain access [env: API_KEY=]
      --ospd-socket <ospd-socket>
          socket to ospd [env: OSPD_SOCKET=]
      --result-check-interval <SECONDS>
          interval to check for new results in seconds [env: RESULT_CHECK_INTERVAL=]
  -l, --listening <IP:PORT>
          the address to listen to (e.g. 127.0.0.1:3000 or 0.0.0.0:3000). [env: LISTENING=]
  -L, --log-level <log-level>
          Level of log messages to be shown. TRACE > DEBUG > INFO > WARN > ERROR [env: OPENVASD_LOG=]
  -h, --help
          Print help
```

# Options

| Option                  | Long Command            | Short Command | Config Section             | Config Name      | Environment Variable  | Description                                                                                                      | Default Value            |
| ----------------------- | ----------------------- | ------------- | -------------------------- | ---------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------ |
| Config Path             | --config                | -c            |                            |                  | OPENVASD_CONFIG       | Path to toml config file                                                                                         |                          |
| Feed Path               | --feed-path             |               | feed                       | path             | FEEED_PATH            | Path to openvas feed                                                                                             | /var/lib/openvas/plugins |
| Feed Check Interval     | --feed-check-interval   |               | feed.check_interval        | secs</br>nanos   | FEED_CHECK_INTERVAL   | Interval to check for feed updates in seconds. Using the config file, it can be set in seconds and nanoseconds   | 3600 (seconds)           |
| TLS Certificates        | --tls-certs             |               | tls                        | certs            | TLS_CERTS             | Path to server TLS certs file. If none is given, TLS is disabled                                                 |                          |
| TLS Key                 | --tls-key               |               | tls                        | key              | TLS_KEY               | Path to server TLS key                                                                                           |                          |
| TLS Client Certificates | --tls-client-certs      |               | tls                        | client_certs     | TLS_CLIENT_CERTS      | Path to client TLS certs enables mTLS                                                                            |                          |
| Enable get scans        | --enable-get-scans      |               | endpoints                  | enable_get_scans | ENABLE_GET_SCANS      | Enables GET /scans endpoint                                                                                      | false                    |
| API key                 | --api-key               |               | endpoints                  | key              | API_KEY               | API key that must be set as X-API-KEY header to gain access. If none is given, api-key authorization is disabled |                          |
| OSPD Socket             | --opsd-socket           |               | ospd                       | socket           | OSPD_SOCKET           | Path to the unix socket of ospd-openvas                                                                          | /var/run/ospd/ospd.sock  |
| Socket read timeout     | --read-timeout          |               | ospd.read_timeout          | secs</br>nanos   | READ_TIMEOUT          | Max time openvasd waits for an ospd-openvas response before returning a 500 code (Internal server error). Using the config file, it can be set in seconds and nanoseconds | Waits forever            |
| Result Check Interval   | --result-check-interval |               | ospd.result_check_interval | secs</br>nanos   | RESULT_CHECK_INTERVAL | Interval to check for new results in seconds. Using the config file, it can be set in seconds and nanoseconds    | 1 (second)               |
| Listening               | --listening             | -l            | listener                   | address          | LISTENING             | IP address and port to listen to                                                                                 | 127.0.0.1:3000           |
| Log Level               | --log-level             | -L            | log                        | level            | OPENVASD_LOG          | Level of log messages to be shown. TRACE > DEBUG > INFO > WARN > ERROR                                           | INFO                     |
| Help                    | --help                  | -h            |                            |                  |                       | Print help                                                                                                       |                          |
