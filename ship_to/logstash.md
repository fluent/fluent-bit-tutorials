# Logstash

## Introduction

[Fluent-bit](https://fluentbit.io) has not an output for Logstash, but we can send records to Logstash by using it [HTTP Output plugin](https://docs.fluentbit.io/manual/output/http) and configuring the [Logstash HTTP input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-http.html) from Logstash side.

## Getting started

### Configuration files

In your `fluent-bit` main configuration file append the following _Output_ section:

```
[OUTPUT]
    Name   http
    Match  *
    Host   192.168.2.3
    Port   12345
    Format json
```

where:
* `Host`: is the IP address or hostname of the target Logstash HTTP input server.
* `Port`: is the TCP port of the target Logstash HTTP input server.
* `Format`: is the communication format. Logstash added support to [msgpack codec](https://www.elastic.co/guide/en/logstash/current/plugins-codecs-msgpack.html), but it seems to be uncompatible with `fluent-bit` format. Due to that, we have to use `json` format to transmit data from `fluent-bit` to `logstash.

In your `logstash` pipeline configuration file, append the following _Input_ and _Filter_ sections:
```
input {
  http {
    port      => 12345
    add_field => { "[@metadata][input-http]" => "" }
  }
}

filter {
  if [@metadata][input-http] {
    date {
      match => [ "date", "UNIX" ]
      remove_field => [ "date" ]
    }
    mutate {
      remove_field => ["headers","host"]
    }
  }
}
```

where:

* `input.http.port`: is the TCP port to bind to.

By default Fluent Bit sends timestamp information on the `date` field, but Logstash expects date information on `@timestamp` field. In order to use `date` field as a timestamp, we have to identify records providing from Fluent Bit. We can do it by adding metadata to records present on this input by `add_field => { "[@metadata][input-http]" => "" }`. Then, we can use the [date](https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html) filter plugin to convert `date` field to `@timestamp`, but only when records provide from this http input (via conditional `if [@metadata][input-http] { ··· }`).

On the other hand, Logstash HTTP input plugin adds to each record information about http requester (fluent-bit in our case). As we don't need this information, we can remove it by using the [mutate plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-mutate.html) and removing fields `headers` & `host` (these fields are customisable on logstash input filter by setting `request_headers_target_field` and `remote_host_target_field` respectively).

### Security

Previous configuration uses plain HTTP communication, which is insecure and can be intercepted. We can increase security by enabling TLS/SSL and adding basic authentication.

In your `fluent-bit` main configuration append the folloging _Output_ section:
```
[OUTPUT]
    Name          http
    Match         *
    Host          192.168.2.3
    Port          12345
    Format        json
    HTTP_User     user
    HTTP_Passwd   password
    tls           On
    tls.verify    Off
```
where:

* `HTTP_User`: is the Basic Auth Username.
* `HTTP_Passwd`: is the Basic Auth Password.
* `tls.*`: corresponds to the [TLS / SSL](https://fluentbit.io/documentation/current/configuration/tls_ssl.html) configuration. In our case, we gonna enable TLS communication without forcing the certificate validation because we gonna use a self-signed certificate. If you want to validate server certificate, configure it according to documentation.

In your `logstash` pipeline configuration file, append the following _Input_ and _Filter_ sections:

```
input {
  http {
    port               => 12345
    add_field          => { "[@metadata][input-http]" => "" }
    user               => "user"
    password           => "password"
    ssl                => true
    ssl_certificate    => "/etc/logstash/http-input.crt"
    ssl_key            => "/etc/logstash/http-input.key"
    ssl_key_passphrase => "ssl_passphtrase"
  }
}

filter {
  if [@metadata][input-http] {
    date {
      match => [ "date", "UNIX" ]
      remove_field => [ "date" ]
    }
    mutate {
      remove_field => ["headers","host"]
    }
  }
}
```

where:

* `input.http.user`: is the user for basic authorization.
* `input.http.password`: is the password for basic authorization.
* `input.http.ssl*`: corresponds to the TLS / SSL configuration. Current Logstash http input version (`v3.2.0` released on `2018-05-10`) only accepts PCK8 certificate format. We can create a valid self-signed certificate via:

```
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out /etc/logstash/http-input.crt -days 365
openssl pkcs8 -in key.pem -topk8 -out /etc/logstash/http-input.key
```
