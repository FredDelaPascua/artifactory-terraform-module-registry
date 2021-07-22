# POC

## Setting up Artifactory PRO

1. To test this plugin you will need to get an Artifactory PRO license, you can get one with the 30-days free trial [here](https://jfrog.com/start-free/#hosted).

You will get an email with the license, we will need that later on.

2. To set up Artifactory PRO with the docker container we will need to download a zip package with some required files to run it, you can download it from [here](https://jfrog.com/download-jfrog-platform/), depending on your OS.

Then, clone the repository and `cd` into the artifactory directory.

    cd artifactory-terraform-module-registry/POC/artifactory

Extract the file in there, it will have two directories `app/` and `var/`, we will need to point the volume for the container to the `var/` directory.

3. We will build and image of Artifactory PRo with the plugin inside.

```docker build -t tfregistry .```

With this `Dockerfile` we are just copying the `terraformModuleRegistry.groovy` inside artifactory on this path `/var/opt/jfrog/artifactory/etc/artifactory/plugins/`. Then we need to run the container.

4. Let's run artifactory. 

```docker run --name artifactory -v $pwd/artifactory/var/:/var/opt/jfrog/artifactory -d -p 8081:8081 -p 8082:8082 tfregistry```

The you can go to localhost:8082 and you should see the jfrog logo starting artifactory. This might take some minutes.

This let us know that is working.

## Setting up apache

First apache needs to be set up as reverse proxy.

1. Go into the artifactory UI at localhost:8082 and log in with user:`admin` password:`password`, there artifactory will guide you to change your password and it will ask you for your license, copy and past it from the email you got into the textbox and continue.
Once you are in artifactory go to the `administration` tab at the left bottom go to `Artifactory` and the `HTTP settings` there set it up:
- Server provider: apache
- Internal hostname: localhost
- Public server name: localhost
- HTTP port: 8082
and the click on download and the save. You will get an `Artifactory.conf` file.

Since you are in the artifactory directory, we need to get into the apache directory.

```cd ../apache/```

Move the file to this location.

Here we already have the configuration files.
What I did in httpd.conf:

* Enable mod_proxy, mod_deflate, mod_proxy_html, mod_proxy_connect, mod_proxy_http, mod_proxy_balancer, mod_lbmethod_byrequests
* Include conf/extra/artifactory.conf
* Include conf/extra/httpd-vhosts.conf

For httpd-vhosts.conf you need to add the redirect routes and copy the rewrite block inside Virtualhost block. 
The redirect routes are pointing to the artifactory's container IP, you can get that with `docker inspect artifactory`.

Also there is a `terraform.json` file that will bbe copied to the root in apache, inside a `.well-known/` directory.

2. Build the image using the Dockerfile, this will copy the new modifyed files inside.

```docker build -t apache .```

3. Run apache as reverse proxy.

```docker run -dit --name proxy -p 80:80 apache```

Now, go to localhost on your nrowser and you should be able to see artifactory from there.

## Enable HTTPS

Terraform requires a HTTPS endpoint for the registry URL, as this poc is running locally we have an extra step to enable it.

1. Go to [ngrok](https://ngrok.com/) and download it.

2. Here are the [instructions](https://dashboard.ngrok.com/get-started/setup) to setup.

3. After running `./ngrok http 80`, you will get a https endpoint redirecting to your localhost:80. Thats the one we are gonna be using. Leave the terminal and open a new one.

## Setting up the consumer 

We are going to set up an ubuntu container and intall terraform to consume our modules.

1. Go to consumer directory.

```cd ../consumer```

2. The important step here is that we have to create a `.terraformrc` file with the artifactory credencials on base64.

```touch .terraformrc```
```echo -n admin:<mysecretpassword> | openssl base64```

Then, the .terraformrc should look like this:


    credentials "fd8cceb630d6.ngrok.io" {
           token = "YWRtaW46bXlzZWNyZXRwYXNzd29yZA=="
    }


3. Build the image

This will install everything that is needed and copy the credentials file to the user root.

```docker build -t tfconsumer .```

4. Now, run the consumer.

```docker run -it --name consumer tfconsumer```

This will leave the terminal open inside the container, there you can make a terraform file to consume the module.

```mkdir terraform && cd terraform```
```touch main.tf```

In main.tf you can now call the module with this references:

    module "whatever" {
        source = "fd8cceb630d6.ngrok.io/<your-local-repo>/<module-name>/<provider>"
        version = ">= 0.1"
    }

## Usage

1. First we need to go to artifactory and create a local repository.

2. Then, wherever you are, we need to create a `terraform-registry` file and upload it to the local-repo, in which the plugin will write.

    touch terraform-registry

    curl -u admin:your-password -XPUT http://localhost:8082/artifactory/local-repo-name/terraform-registry -T terraform-registry

3. Let's upload your module.

Move to the directory on where you have your module, and you will need to compress the files into a `.tar.gz` file.

    tar -czvf <module-name>.tar.gz -C <module-name> .

Then upload it.

    curl -u admin:<password> -XPUT http://localhost:8082/artifactory/<local-repo-name>/<module-name.tar.gz> -T <module-name.tar.gz>

4. In artifactory go to the `administration` tab, al the left bottom go to Artifactory and the `Property set`.

Create a property set named `terraform`, and add the next properties:

* module.name
* module.provider
* module.version

Then at the left side go to Repositories > Repositories, choose your repository and go to it's advance tab and move your new property set to the right.

Afterwards go to you local repository, choose you `module-name`.tar.gz, pick your new property set and then set the values of each property.


This should have evrything working now, go back to your consumer terminal and run `terraform init` to test it.


