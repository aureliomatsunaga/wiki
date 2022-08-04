Karafka is currently being used in production with the following deployment methods:

  - systemd (+ Capistrano)
  - Docker
  - Heroku
  - AWS with MSK ([Fully Managed Apache Kafka](https://aws.amazon.com/msk/))

Since the only thing that is long-running is the Karafka server, it shouldn't be hard to make it work with other deployment and CD tools.

## systemd (+ Capistrano)

You can easily manage Karafka applications with `systemd`. Here's an example `.service` file that you can use.

```bash
# Move to /lib/systemd/system/karafka.service
# Run: systemctl enable karafka

[Unit]
Description=karafka
After=syslog.target network.target

[Service]
Type=simple

WorkingDirectory=/opt/current
ExecStart=/bin/bash -lc 'bundle exec karafka server'
User=deploy
Group=deploy
UMask=0002

RestartSec=1
Restart=on-failure

# output goes to /var/log/syslog
StandardOutput=syslog
StandardError=syslog

# This will default to "bundler" if we don't specify it
SyslogIdentifier=karafka

[Install]
WantedBy=multi-user.target
```

If you want to use `systemd` based solution together with Capistrano, you don't need the `capistrano-karafka` gem. Instead, you can use this simple Capistrano `.cap` file:

```ruby
# frozen_string_literal: true

after 'deploy:starting', 'karafka:stop'
after 'deploy:published', 'karafka:start'
after 'deploy:failed', 'karafka:restart'

namespace :karafka do
  task :start do
    on roles(:app) do
      execute :sudo, :systemctl, :start, 'karafka'
    end
  end

  task :stop do
    on roles(:app) do
      execute :sudo, :systemctl, :stop, 'karafka'
    end
  end

  task :restart do
    on roles(:app) do
      execute :sudo, :systemctl, :restart, 'karafka'
    end
  end

  task :status do
    on roles(:app) do
      execute :sudo, :systemctl, :status, 'karafka'
    end
  end
end
```

If you need to run several processes of a given type, please refer to `template unit files`.

## Docker

Karafka can be dockerized as any other Ruby/Rails app. To execute ```karafka server``` command in your Docker container, just put this into your Dockerfile:

```bash
ENV KARAFKA_ENV production
CMD bundle exec karafka server
```

## AWS + MSK (Fully Managed Apache Kafka)

First of all, it is worth pointing out that Karafka, similar to librdkafka does **not** support SASL mechanism for AWS MSK IAM that allows Kafka clients to handle authentication and authorization with MSK clusters through [AWS IAM](https://aws.amazon.com/iam/). This mechanism is a proprietary idea that is not part of Kafka.

Karafka **does**, however, support standard SASL + SSL mechanisms. Please follow the below instructions for both cluster initialization and Karafka configuration.

### AWS MSK cluster setup

1. Navigate to the AWS MSK page and press the `Create cluster` button.
2. Select `Custom create` and `Provisioned` settings.

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/creation_method.png" />
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/provisioned2.png" />
</p>

3. Use custom config and set `auto.create.topics.enable` to `true` unless you want to create topics using Kafka API. You can change it later, and in general, it is recommended to disallow auto-topic creation (typos, etc.), but this can be useful for debugging.

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/broker-settings.png" />
</p>

4. Setup your VPC and networking details.
5. Make sure that you **disable** the `Unauthenticated access` option. With it enabled, there won't be any authentication beyond those imposed by your security groups and VPC.
6. **Disable** `IAM role-based authentication`.
7. **Enable** `SASL/SCRAM authentication`

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/access-methods.png" />
</p>

8. Provision your cluster.
9. Make sure your cluster is accessible from your machines. You can test it by using the AWS VPC Reachability Analyzer.

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/reachability.png" />
</p>

10. Visit your cluster `Properties` page and copy the `Endpoints` addresses.

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/brokers-list.png" />
</p>

11. Log in to any of your machines and run a `telnet` session to any of the brokers:
```bash
telnet your-broker.kafka.us-east-1.amazonaws.com 9096

Trying 172.31.22.230...
Connected to your-broker.kafka.us-east-1.amazonaws.com.
Escape character is '^]'.
^Connection closed by foreign host.
```

If you can connect, your settings are correct, and your cluster is visible from your instance.

12. Go to the AWS Secret Manager and create a key starting with `AmazonMSK_` prefix. Select `Other type of secret` and `Plaintext` and provide the following value inside of the text field:

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/secret_type2.png" />
</p>

13. In the `Encryption key` section, press the `Add new key`.

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/add_new_key.png" />
</p>

14. Create a `Symmetric` key with `Encrypt and decrypt` as a usage pattern.

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/keys.png" />
</p>

15. Select your key in the `Encryption key` section and press `Next`.
16. Provide a secret name and description and press `Next` until you reach the `Store` button.
17. Store your secret.
18. Go back to the AWS MSK and select your cluster.
19. Navigate to the `Associated secrets from AWS Secrets Manager` section and press `Associate secrets`

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/associate_secrets.png" />
</p>

20. Press the `Choose secrets` and select the previously created secret.

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/associate_secrets2.png" />
</p>

21. Press `Associate secrets`. It will take AWS a while to do it.
22. Congratulations, you just configured everything needed to make it work with Karafka.

### Karafka configuration for AWS MSK SASL + SSL

Provide the following details to the `kafka` section:

```ruby
config.kafka = {
  'bootstrap.servers': 'yourcluster-broker1.amazonaws.com:9096,yourcluster-broker2.amazonaws.com:9096',
  'security.protocol': 'SASL_SSL',
  'sasl.username': 'username',
  'sasl.password': 'password',
  'sasl.mechanisms': 'SCRAM-SHA-512'
}
```

After that, you should be good to go.

### Troubleshooting AWS MSK

#### Local: Authentication failure

```
ERROR -- : rdkafka: [thrd:sasl_ssl://broker1.kafka.us-east-1.amazonaws.]:
sasl_ssl://broker1.us-east-1.amazonaws.com:9096/bootstrap: SASL authentication error:
Authentication failed during authentication due to invalid credentials with SASL mechanism SCRAM-SHA-512
(after 312ms in state AUTH_REQ, 1 identical error(s) suppressed)

ERROR -- : librdkafka internal error occurred: Local: Authentication failure (authentication)

```

It may mean two things:
- Your credentials are wrong
- AWS MSK did not yet refresh its allowed keys, and you need to wait. Despite AWS reporting cluster as `Active` with no pending changes, it may take a few minutes for the credentials to start working.

#### Connection setup timed out in state CONNECT

```
rdkafka: [thrd:sasl_ssl://broker1.kafka.us-east-1.amazonaws.]:
sasl_ssl://broker1.us-east-1.amazonaws.com:9092/bootstrap:
Connection setup timed out in state CONNECT (after 30037ms in state CONNECT)
```

This means Kafka is unreachable. Check your brokers' addresses and ensure you use a proper port: `9096` with SSL or `9092` when plaintext. Also, make sure your instance can access AWS MSK at all.

### Connection failures and timeouts

Please make sure that your instances can reach Kafka. Keep in mind that security group updates can have a certain lag in propagation.

### Rdkafka::RdkafkaError (Broker: Invalid replication factor (invalid_replication_factor))

Please make sure your custom setting `default.replication.factor` value matches what you have declared as `Number of zones` in the `Brokers` section:

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/instructions/msk/brokers_count.png" />
</p>