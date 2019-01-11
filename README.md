# fluentd helm chart for kubernetes 

[![Build Status][14]](https://jenkins.migrations.cnct.io/job/pipeline-fluentd/job/master)

[Fluentd][1] is an open source data collector, which lets you unify data collection and consumption for a better use and understanding of data.
This chart is meant to be used with [this fluent-bit chart][2] on a kubernetes cluster. Fluent-bit will be deployed as a daemonset using [this image][3] to collect all logs and will then push them to fluentd using the [in_forward protocol][4], secured by TLS. FLuentd will pass on logs to [elasticsearch][5], secured by [xpack][6], and to [s3][7] for long term archival. 

### Forward logs to elasticsearch 
This system sends on all logs to [elasticsearch][11]. To connect correctly, username and password fields will need to be set to allow fluentd to communicate securely with [elasticsearch xpack][6]. These fields will end up becoming the `user` and `password` fields of the elasticsearch output plugin configuration. They should match an elasticsearch xpack username and password and can be passed in at installation in with: 
```
--set fluentESUser=your-username,fluentESPassword=your-password
```

### Log Archival using s3 
Fluentd has an output plugin to push logs to [s3][7]. This plugin must be included in the `values.yaml` file under the `plugins` section to be installed.  You must first go into s3 and create the bucket and path that you wish to push to. Then either use or create an IAM role that has s3 read/write access. 

Several pieces of information must be passed in at chart install for this to work properly, as the plugin needs access to your aws credentials, and needs to know which region and bucket you would like to push the logs to. 
Make sure the following envs are set: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` 

You will also need to provide s3 bucket name and region at chart installation. 
For example: `s3Bucket=myLogs` and `s3Region=us-west-2`

### Secure forward of fluentbit logs using TLS 
This feature is turned off by default. To enable it, uncomment the `<transport tls>` section of the `forward-input.conf` configmap in `values.yaml`.  Comment out the `bind` field, and turn the `enableTLS` value in the same file to `true`.
Once enabled, client and server certs must be created. For a quick walkthrough of how this can work with fluent-bit and fluentd, read [this blog][10]
For fluent-bit to connect and securely send logs to fluentd, TLS information must be passed in as a [kubernetes secret][13] to the cluster before this chart can be deployed. 
```
kubectl create secret generic fluentd-tls \
  --from-file=ca.crt.pem=./certs/ca.crt.pem \
  --from-file=server.crt.pem=./certs/server.crt.pem \
  --from-file=server.key.pem=./private/server.key.pem
```

It is recommended to protect keys with a password, although this step can be skipped if desired. Set the fluentD passphrase at installation with 
```
--set fluentdPrivateKeyPassphrase=your-ssl-passphrase
```
  
Ensure that client certs are passed into secrets as well and used by fluent-bit. 

## To install fluentd from local repository 
```
helm install ./fluentd/ --name fluentd --namespace=your-namespace --set awsKeyId="$AWS_ACCESS_KEY_ID",awsSecKey="$AWS_SECRET_ACCESS_KEY",s3Bucket=your-bucket,s3Region=your-region,fluentdPrivateKeyPassphrase=your-ssl-passphrase,fluentESUser=your-username,fluentESPassword=your-password
``` 

### To use other output plugins or remove s3 or elasticsearch
- search for [plugins][12] 
- create a yaml file to hold new values  
- include desired plugin in `plugins` section of file if necessary, or take out plugins not needed 
- update `output.conf` to reflect desired plugin usage 
- when installing chart, pass in new file using `-f <new_values.yaml>`  
``` 
example installation: 
helm install ./fluentd/ --name fluentd -f <new_values.yaml>
```

#### Resources: 
- [fluentd s3 documentation][8]
- [s3 plugin source code][9]

[1]: https://www.fluentd.org/
[2]: https://github.com/samsung-cnct/chart-fluent-bit
[3]: https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/fluentd-elasticsearch/fluentd-es-image/Dockerfile
[4]: https://docs.fluentd.org/v1.0/articles/in_forward
[5]: https://github.com/samsung-cnct/chart-elasticsearch
[6]: https://www.elastic.co/guide/en/elastic-stack-overview/current/elasticsearch-security.html
[7]: https://aws.amazon.com/s3/
[8]: https://docs.fluentd.org/v1.0/articles/out_s3
[9]: https://github.com/fluent/fluent-plugin-s3
[10]: https://banzaicloud.com/blog/k8s-logging-tls/
[11]: https://www.elastic.co/products/elasticsearch
[12]: https://www.fluentd.org/plugins
[13]: https://kubernetes.io/docs/concepts/configuration/secret/
[14]: https://jenkins.migrations.cnct.io/buildStatus/icon?job=pipeline-fluentd/master



