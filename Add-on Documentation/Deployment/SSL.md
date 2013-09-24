# SSL Add-on

SSL encryption is available for improved security when transmitting passwords
and other sensitive data. As part of the provided `.cloudcontrolled.com`
subdomain all deployments have access to piggyback SSL using a
`*.cloudconrolled.com` wildcard certificate. To use this, simply point your
browser to `https://APP_NAME.cloudcontrolled.com` for the default deployment or
to `https://DEP_NAME-APP_NAME.cloudcontrolled.com` for non-default deployments
(please note the dash between DEP_NAME and APP_NAME). SSL support for custom
domains is available through the SSL add-on.

## Custom Domain Certificates

To enable SSL support for custom domains like `www.example.com` or
`secure.example.com` you need the SSL add-on. Root or naked domains like
`example.com` are not supported.

Please follow these three simple steps to add SSL support to your deployment.

 1. Acquire a signed certificate from your certificate authority of trust.
 2. Add the SSL addon providing the certificate, the private key and
    certificate-chain files.
 3. Set your DNS entry to point to your SSL DNS Domain

Note: Please allow up to one hour for DNS changes to propagate before they go
into effect.

### Acquiring an SSL Certificate

The is wide variety of Certificate Authorities (CA) which differ in the cost
and the process of acquiring an SSL certificate. The majority of them offers a
trial period that you can try and compare every service. In most cases you need
to go through the following steps.

Note: For testing purposes you can always use a self-signed certificate which
is free of costs and does not require going through the registration process
of individual providers.

#### Generate a private key

For acquiring an SSL Certificate you need to provide to your CA an RSA Private
Key and a CSR (Certificate Singing Request). Those can also be used for
creating self-signed certificates. To generate them you need the openssl
toolkit which can be install with one of the following ways according to your
platform.

|Platform|Install method|
|:-------|:-------------|
|Mac OS X|[Homebrew](http://mxcl.github.com/homebrew/):`brew install openssl`|
|Windows|[Windows package installer](http://gnuwin32.sourceforge.net/packages/openssl.htm)|
|Ubuntu Linux|`apt-get install openssl`|

After you are done with the installation use the openssl command line tool to
create your private RSA key:
 ~~~
 $ openssl genrsa -des3 -out server.key.org 2048
 # Enter and confirm a password
 ~~~

The generated key is protected by a password which needs to be removed so
that it can be loaded by the web server without asking for the password.

 ~~~
 $ openssl rsa -in server.key.org -out server.key
 ~~~

Your private key used for the process is now saved in the file `server.key`

#### Generate a CSR (Certificate Singing Request)

The next set is generating the CSR which is required by the CA and is
containing all the information regarding your company or organization thus
prompting you to enter those.

 ~~~
 $ openssl req -new -key server.key -out server.csr
 # Output
 # Country Name (2 letter code) [AU]:DE
 # State or Province Name (full name) [Some-State]:
 # Locality Name (eg, city) []:
 # Organization Name (eg, company) [Internet Widgits Pty Ltd]:
 # Organizational Unit Name (eg, section) []:Information Technology
 # Common Name (eg, your name or your server's hostname) []:www.example.com
 # Email Address []:
 # Please enter the following 'extra' attributes
 # to be sent with your certificate request
 # A challenge password []:
 # An optional company name []:
 ~~~

The file created after this process is a `server.csr`.

Please pay attention to the fields Country Name and Common Name. The country
name should contain the 2 letter code of your country according to the
[ISO 3166-1](http://www.iso.org/iso/country_codes/iso_3166_code_lists/country_names_and_code_elements.htm)
format. Second and most important is the common name. That should reflect the
domain for which you want to issue the certificate. As mentioned earlier this
cannot be a root domain but has to have a format like `www.example.com`.

#### Issuing the Certificate

After choosing you CA you have to go through their process of issuing the
certificate. For that you will need to provide the CSR file and in many cases
to define the web server you are going to use. In that case you should select
the Nginx web server and if this is not an option then Apache 2.x should also
be OK.

In the end your CA will provide you some files including the SSL certificate
and the Certificate Chain.

Note: The certificate chain is a chain of trust that proves that your
certificate is issued by a trustworthy provider authorized by a Root CA.  Root
CA certificates are stored in all modern browsers and this is how your browser
is able to verify that your website is secure. In the other case you will get
something like this
![Firefox warning](http://www.nczonline.net/blog/wp-content/uploads/2012/08/ffssl.png)

### Adding the SSL addon

To add the SSL addon simply provide the paths to the files provided by the
certificate authority using the respective parameters of the addon.add command.

 ~~~
 $ cctrlapp APP_NAME/DEP_NAME addon.add ssl.host --cert path/to/CERT_FILE --key path/to/KEY_FILE --chain path/to/CHAIN_FILE
 ~~~

In order to check the status of the addon you can do the following.

 ~~~
 $ cctrlapp APP_NAME/DEP_NAME addon ssl.host
 ~~~

When the SSL certificate is expired, you can update it by removing the addon
and re-adding it providing the updated certificate. The SSL service is provided
for 23 minutes after removing the addon so that it can be updated in the
meantime without interrupting the service. To achieve that you have to run the
following commands.

 ~~~
 $ cctrlapp APP_NAME/DEP_NAME addon.remove ssl.host
 $ cctrlapp APP_NAME/DEP_NAME addon.add ssl.host --cert path/to/NEW_CERT_FILE --key path/to/KEY_FILE --chain path/to/CHAIN_FILE
 ~~~

Note: You need provide the original key and chain again when updating the
addon even if those are not changed.

## HTTPS Redirects

HTTPS termination is done at the routing tier. Requests are then routed via
HTTP to one of your app's clones. To determine if a request was made via HTTPS
originally the routing tier sets the `X-FORWARDED-PROTO` header to `https`. The
header is only set for requests that arrived via HTTPS at the routing tier.
This allows you to redirect accordingly.

### PHP Example

For PHP you can either redirect via Apache's mod_rewrite using a `.htaccess`
file or directly in your PHP code.

#### .htaccess
~~~
<IfModule mod_rewrite.c> 
    RewriteEngine On

    RewriteCond %{HTTP:X-FORWARDED-PROTO} !=https [NC]
    RewriteRule ^.*$ https://%{HTTP_HOST}
</IfModule>
~~~

#### PHP
~~~php
<?php
    if (!isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && 
        $_SERVER['HTTP_X_FORWARDED_PROTO'] != 'https') {
        header(
            'Location: https://' . 
            $_SERVER['HTTP_HOST'] . 
            $_SERVER['REQUEST_URI']
        );
    }
?>
~~~
