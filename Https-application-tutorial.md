# Create an https webapp running on local host

## Generate ssl keys using openssl

The detailed steps are described [here](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu-16-04). 

You can just run this command: it will generate the files you need.

    mkdir encryption
    cd encryption
    openssl req -newkey rsa:2048 -nodes -keyout domain.key -x509 -days 365 -out domain.crt

Answer the questions you are asked the way you want, these keys are temporary anyway.

## Configure a node.js application to use the generated keys

We have an application running on express.js and we want to make it a https application.
You can use the https module you can install it very simply:

    npm install https
    npm install fs

You can import this module as you usually do with any module:

    const https = require('https');
    const fs = require('fs'); //file reading module needed to read the generated ssl keys

Read the keys you generated:

    const key = fs.readFileSync('encryption/private.key');
    const cert = fs.readFileSync('encryption/certificate.crt');
    const ca = fs.readFileSync('encryption/ca_bundle.crt');

    const options = {
      key,
      cert,
      ca = cert, //this is a cheat trick, you don't need authentification for now, just user yourself as Certification Authority
    };

Kindly ask https to listen on an open port and redirect the requests to your app.
    
    port = 1234; // you don't have to choose the same as me ;)
    // app.listen(port); // comment out this line if you don't want to serve non https requests anymore
    https.createServer(options, app).listen(port); // https will decrypt the requests and transfer them to your app.

Then your app is listening to the port you specified:

    node app.js

Go to your browser and don't forget to ask for the https page ;)
    https://localhost:1234


# Deploy https server with nginx

If you are using nginx, you need to configure it to use your ssl keys.
This [tutorial](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu-16-04) contains all the steps.

First, if you tested the https on your localhost, use an encrypted method to copy the "temporary ssl keys" on the remote server, (ssh or scp is fine).


## Preparation

Edit this file /etc/nginx/snippets/self-signed.conf, you need super user priviledges.
Set the values of ssl_certificate and ssl_certificate_key variables.

    ssl_certificate /path/to/your/encryption/folder/encryption/domain.crt;
    ssl_certificate_key /path/to/your/encryption/folder/encryption/domain.key;
    
Make a backup of the nginx configuration file:
    
    cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak

## Configure nginx to redirect your traffic to https:

Edit the /etc/nginx/sites-available/default file (root privileges needed).

redirect to https

    server {
            listen 80 default_server;
            listen [::]:80 default_server;
            server_name _;
            # Tell nginx to redirect the traffic to the same url with https instead of http
            return 301 https://$host$request_uri;
    }

    server {
            # SSL configuration
            #
            listen 443 ssl default_server;
            listen [::]:443 ssl default_server;
            # tell nginx where to find the 
            include snippets/self-signed.conf;

            # don't forget to check that you are redirecting to the *https* protocol
            location / {
                # [...]
                proxy_pass https://localhost:8080; 
                # [...]
            }
    }

Check nginx configuration and restart it:

    nginx -t
    systemctl restart nginx


# Use sslforfree to generate authenticated certificate

This website provides free authenticated ssl keys: [https://www.sslforfree.com](https://www.sslforfree.com).

Enter the domain name you want to authenticate, then click "create free ssl certificate".

The website is now asking you to "prove" that you are the webmaster of this domain.

If you did not setup a ftp server, then the "Manual Verification" (in the middle) is the method you want.

Download the file they generated for you and save it as, sslforfree-challenge.bin.

Now we need to hack the node.js application so that it answers the content of that file at the url that appears bellow (something with your.domain.com/.well-known/acme-challenge/[...]).

If you are using express.js and fs:

Download the content of the file using fs:

    const sslforfree = fs.readFileSync('path/to/sslforfree-challenge.bin');

Kindly ask your app to perform the challenge.

    app.get('/.well-known/acme-challenge/*type-generated-path-here*', (req, res) => {
      res.send(sslforfree);
    });

Save and restart your app on the server.

It is recommended to try to manually connect to the url before clicking the "Dowload SSL Certificate" button, because the "challenge" changes everytime you fail it.
If your app is answering well the browser should ask you to download the file you just got from sslforfree platform.

If you succeed the challenge, you can download the three files containing your authenticated keys.
Replace your "self certified" keys by those you just downloaded.

You node.js application source code may look like this:

    const key = fs.readFileSync('encryption/private.key');
    const cert = fs.readFileSync('encryption/certificate.crt');
    const ca = fs.readFileSync('encryption/ca_bundle.crt');

    const options = {
      key,
      cert,
      ca,
    };


Don't forget to update nginx configuration as well.

Et voilà, restart everything, and your website is certified. 
