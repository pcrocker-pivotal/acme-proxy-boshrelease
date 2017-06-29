### Intro

This bosh release allows user to do multiple things. It can be used for various scenarios:

 - Running Nginx as reverse proxy and terminate multiple TLS connections.
 - Standalone errand to renew Letsencrypt certificate by trigger an errand (Only support HTTP acme challenge)
 - Standalone errand to import certificates and private keys to credhub
 - A combined errand to renew Letsencrypt certificate and import of new certs/keys to credhub db.

This BOSH release is built to be used together with the community [nginx-release](https://github.com/cloudfoundry-community/nginx-release).

1. Upload the release to BOSH director

```
bosh upload-release https://github.com/govau/nginx-tlsconfig-release/releases/download/1.1/release_v1.1.tgz
```

2. Create BOSH manifest to deploy nginx server using both [nginx-release](https://github.com/cloudfoundry-community/nginx-release) and this release.

Check the manifest in `examples/` subdirectory as a template.

To build a release tarball yourself and upload onto bosh, you can run the following command.

```
git clone https://github.com/govau/nginx-tlsconfig-release.git
bosh2 create-release --tarball=release.tgz
bosh2 -e vbox upload-release release.tgz
```


### Example with boshlite

Add `10.244.0.10` to the list of static ips for "default" network:

 - Execute `bosh2 -e vbox cloud-config > cloud-config.yml` to acquire latest cloud-config
 - Modify cloud-config file and add 10.244.0.10 to the list of static ip (or create a new entry) as shown below
 - Execute `bosh2 -e vbox update-cloud-config cloud-config.yml` to update the cloud-config

```
networks:
- name: default
  subnets:
  - azs:
    - z1
    - z2
    - z3
    cloud_properties:
      name: vboxnet2
    dns:
    - 8.8.8.8
    gateway: 10.244.0.1
    range: 10.244.0.0/16
    reserved:
    - 10.244.0.0-10.244.0.4
    static:
    - 10.244.0.10
```
Deploy the example deployment manifest with an operator file which specify information about the new cert (this.example.com)

```
bosh2 -e vbox -d nginx deploy ./example/nginx_manifest.yml -o ./example/this_example_com.yml
```

`bosh2 -e vbox -d nginx instances` should show nginx-ubuntu machine running and nothing happens for errand machine.

To execute the errand, we will need to run `bosh2 -e vbox -d nginx run-errand letsencrypt-errand`

Here is an example to add another domain, we will make a copy of the operator file for this.example.com and do a few `sed` commands to replace the variable name and domain. Example below will add a subdomain that.example.com config

```
cp ./example/this_example_com.yml ./example/that_example_com.yml
sed -i 's/this_example_com/that_example_com/g' ./example/that_example_com.yml
sed -i 's/this\.example\.com/that\.example\.com/g' ./example/that_example_com.yml
```

We can now redeploy the deployment and see if everything working as expected and errand job runs accordingly

```
bosh2 -e vbox -d nginx deploy ./example/nginx_manifest.yml -o ./example/this_example_com.yml -o ./example/that_example_com.yml
bosh2 -e vbox -d nginx run-errand letsencrypt-errand
```

This time we should see the Stdout of the errand job showing 2 attempts to renew certificate, one for this_example_com and another for that_example_com.

### Issues and Lessons

Below is some issues I ran into as well as some notes that may help others when you want to build your own bosh release.

__/bin/run__: If you have an errand instance with a list of jobs on it, the first job in the array HAVE TO have a `/bin/run` or `/bin/run.sh` script which gets trigger when you perform `bosh2 run-errand` command. ONLY this `/bin/run` script is triggered and all other `/bin/run` scripts in other jobs will be ignored.

__/bin/prestart__: All pre-start script for all jobs run before monit job and errand `/bin/run`.

`bosh2 -e vbox -d nginx variables` will show all variables and its credhub associated ID for debugging.

Letsencrypt rate limiting: This took me longer to realized than it should and wasted a good 2, 3 hours as things were working started breaking for no apparent reason and getting pre-start script log from bosh was painful. There is a rate limit on letsencrypt and can be found (here)[https://letsencrypt.org/docs/rate-limits/].

To declare a certificate variable and have it automatically generated by credhub, you are required to at least specify 2 options (ca and common_name - duh). This was not clear in the documentation and previously, using var-store you can actually just ignore these options when doing some testing.

```
variables:
- name: example-ca
  type: certificate
  options:
    is_ca: true
    common_name: example.com
- name: this_example_com
  type: certificate
  options:
    # For credhub to work, We need minimum the common_name and a reference to a CA variable.
    ca: "example-ca"
    common_name: "this.example.com"
```

__Some notes for creating a bosh release:__

Often, in packaging script, you will see the $BOSH_INSTALL_TARGET environment variable being used. For example, below is the one for credhub.

```
tar xzvf credhub/credhub-linux-1.0.0.tgz -C credhub/
mkdir -p $BOSH_INSTALL_TARGET/bin
cp credhub/credhub $BOSH_INSTALL_TARGET/bin/credhub
chmod +x $BOSH_INSTALL_TARGET/bin/credhub
```

This script basically will unpack the credhub blob and copy the binary to machine and become available for use at /var/vcap/packages/credhub/bin/credhub


To add a binary blob to project using S3 bucket - The owner of the release is also in charged of managing the blobstore for that release.:

- Run `bosh2 add-blob  PATH_TO/credhub-linux-1.0.0.tgz credhub-linux-1.0.0.tgz` inside the release root folder. This will create an entry in `./config/blobs.yml` with name of blob, size and its sha1 hash
- Add `./config/private` with the following content
```
---
blobstore:
  options:
    access_key_id: ACCESS_KEY_FOR_S3
    secret_access_key: SECRET_KEY_FOR_S3
```

- In `./config/final.yml` we add the blobstore provider (s3) , the region and the bucket name for this project's blobstore
```
---
name: nginx-tlsconfig-release
blobstore:
  provider: s3
  options:
    region: ap-southeast-2
    bucket_name: nginx-tlsconfig-public-access
```

Now we can upload the blobstore using `bosh2 upload-blobs` command. This will create an object id in ./config/blobs.yml for the binary blob. This id is also the name of the S3 object which you can download at BUCKET_NAME.s3.amazonaws.com/OBJECTID

One last thing about erb template file, if the variable does not have a default value in `spec` and not set in the manifest, the value of it is `nil` (ruby version of null)




__NOTE:__ the operator file for each of these domains do not only have the configuration used for LetsEncrypt certificate signing request (CSR) but also include the virtual host configuration for nginx reverse proxy servers itself. This configuration allow administrators to add custom Headers, Security options , Blacklist URI etc...