FROM httpd:2.4
COPY ./my-httpd.conf /usr/local/apache2/conf/httpd.conf
COPY ./httpd-vhosts.conf /usr/local/apache2/conf/extra/httpd-vhosts.conf
COPY ./artifactory.conf /usr/local/apache2/conf/extra/artifactory.conf
COPY ./terraform.json /usr/local/apache2/htdocs/.well-known/terraform.json
