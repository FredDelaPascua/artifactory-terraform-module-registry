FROM ubuntu:18.04
RUN apt update
RUN apt install -y wget sudo
RUN sudo apt-get install unzip
RUN https://www.terraform.io/downloads.html
RUN wget https://releases.hashicorp.com/terraform/0.12.7/terraform_0.12.7_linux_amd64.zip
RUN unzip terraform_0.12.7_linux_amd64.zip
RUN sudo mv terraform /usr/local/bin/
RUN terraform --version 
COPY ./.terraformrc ~/.terraformrc