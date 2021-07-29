# POC
​
## Setting up Artifactory PRO
​
### Prerequisites 
To set up an Artiffactory PRO you need the following:
​
* Artifactory PRO license:  You can get one with the [30-days free trial](https://jfrog.com/start-free/#hosted), and it is necessary to test this plugin.
​
	**Note:** You will get an email with the license; save it for later.
​
* Zip package from [JFrog Platform](https://jfrog.com/download-jfrog-platform/) with some required files to run.
​
### Process
To set up Artifactory PRO with the docker container:
​
1. Clone the repository and cd into the Artifactory directory.    

        `cd artifactory-terraform-module-registry/POC/artifactory`
				
1. Extract the file in there, it has two directories `app/` and `var/`. 
1. Point the volume for the container to the `var/` directory.
1. Build an image of Artifactory PRO with the plugin inside.

		`docker build -t tfregistry .`
​

**Note:** This Dockerfile just copies the `terraformModuleRegistry.groovy` inside artifactory on this path: `/var/opt/jfrog/artifactory/etc/artifactory/plugins/`
1. Run the container.
1. Run the Artifactory.

		`docker run --name artifactory -v $pwd/artifactory/var/:/var/opt/jfrog/artifactory -d -p 8081:8081 -p 8082:8082 tfregistry`
1. Go to `localhost:8082` and see the Jfrog logo starting Artifactory to know that it is working. 
	
	**Note:** It might take some minutes.
​
## Setting up Apache
​
**Important:** Set up Apache as reverse proxy.

​
To set up Apache:
​
1. Go into the artifactory UI at localhost:8082.
1. Log in with:
	* user: `admin`
	* password: `password`
1. Change your password.
1. Enter your license; you can copy and paste it from the email.
1. Once you are in Artifactory, go to the `administration` tab at the left bottom go to `Artifactory` and the `HTTP settings`.
1. Set up the following information:
	* Server provider: apache
	* Internal hostname: localhost
	* Public server name: localhost
	* HTTP port: 8082
1. Click **Download** and then **Save**. You will get an `Artifactory.conf` file.
1. Get into the apache directory because you are in the Artifactory directory, go to:

        `cd ../apache/`

1. Move the file to this location.
​
​
You already have the configuration files. For example, what I did in the `httpd.conf` file:
​
1. Enable mod_proxy, mod_deflate, mod_proxy_html, mod_proxy_connect, mod_proxy_http, mod_proxy_balancer, mod_lbmethod_byrequests
1. Include conf/extra/artifactory.conf
1. Include conf/extra/httpd-vhosts.conf
1. For httpd-vhosts.conf, add the redirect routes and copy the rewrite block inside Virtualhost block. 
	
**Note:** The redirect routes are pointing to the artifactory's container IP, you can get that with `docker inspect artifactory`.
​
1. Check that the `terraform.json` file is copied to the root in apache, inside a `.well-known/` directory.
​
1. Build the image using the Dockerfile; it copies the newly modified files inside.
​

        `docker build -t apache .`
​
1. Run apache as a reverse proxy.
    
        `docker run -dit --name proxy -p 80:80 apache`
​
1. Go to localhost on your browser, and you should be able to see artifactory from there.
​
## Enabling HTTPS
​
Terraform requires a HTTPS endpoint for the registry URL, as this POC is running locally, we have an extra step to enable it. 

To enable HTTPS:
​
1. Download from [ngrok](https://ngrok.com/) 
1. Follow the [instructions](https://dashboard.ngrok.com/get-started/setup) to set up.
1. Run `./ngrok http 80`
1. Check that you got an HTTPS endpoint redirecting to your `localhost:80`. That is the one you use.
1. Leave the terminal and open a new one.
​
## Setting up the consumer 
​
To set up an ubuntu container and intall Terraform to consume our modules.
​
1. Go to consumer directory.

        `cd ../consumer`

1. Create a `.terraformrc` file with the artifactory credentials on base64.
​
	
        touch .terraformrc
		
	    echo -n admin:<mysecretpassword> | openssl base64	
	
1. Check that the `.terraformrc` looks like this:
​
	
	    credentials "fd8cceb630d6.ngrok.io" {
	           token = "YWRtaW46bXlzZWNyZXRwYXNzd29yZA=="
	    }
	
​
1. Build the image. This will install everything that is needed and copy the credentials file to the user root.
    
        `docker build -t tfconsumer .`
​
1. Run the consumer.
    
        `docker run -it --name consumer tfconsumer`
​
1. Running the consumer will leave the terminal open inside the container, make a terraform file to consume the module.
​
	```
    mkdir terraform && cd terraform

    touch main.tf
	```
1. In main.tf, call the module with it's references:
	```
    module "whatever" {
        source = "fd8cceb630d6.ngrok.io/<your-local-repo>/<module-name>/<provider>"
        version = ">= 0.1"
    }
	```
## Usage
​
To make use of the Artifactory:
​
1. Go to artifactory and create a local repository.
​
1. Create a `terraform-registry` file.
1. Upload it to the local-repo, in which the plugin writes.
​
	```
	touch terraform-registry
	
	curl -u admin:your-password -XPUT http://localhost:8082/artifactory/local-repo-name/terraform-registry -T terraform-registry
	```
1. Upload your module.
	1. Move to the directory where you have your module.
	1. Compress the files into a `.tar.gz` file.

	```
    tar -czvf <module-name>.tar.gz -C <module-name> .
	```
	1. Upload it.
	
	```
    curl -u admin:<password> -XPUT http://localhost:8082/artifactory/<local-repo-name>/<module-name.tar.gz> -T <module-name.tar.gz>
	```
1. In artifactory, go to the `administration` tab, at the left bottom.
1. Go to Artifactory and the Property set.
1. Create a property set named `terraform`, and add the next properties:

	* module.name
	* module.provider
	* module.version
​
1. On the left side, go to Repositories > Repositories, choose your repository.
1. Go to **Advance** tab and move your new property set to the right.
1. Go to your local repository.
1. Choose your `module-name.tar.gz`, pick your new property set.
1.  Set the values of each property.
1. This should have everything working now, go back to your consumer terminal and run `terraform init` to test it.