(copied from Wiki, with additional notes prefixed with MM2019:)

# HTTPS certificates for your Synology NAS using acme.sh

Since Synology introduced [Let's Encrypt](https://letsencrypt.org/), many of us benefit from free SSL. 

On the other hand, many of us don't want to expose port 80/443 to the Internet, including opening ports on the router. The alternative is to use the DNS-01 protocol. Sadly the Synology implementation of Let's Encrypt currently (1-Jan-2017) only supports the HTTP-01 method which requires exposing port 80 to the Internet. Also, if the domain of your NAS has an IPv6 AAAA record set, the Synology implementation of Let's Encrypt will fail.

But we can access the NAS via SSH and configure it to renew certs instead of using the web dashboard.

The following guide will use the DNS-01 protocol using the [Cloudflare API](https://api.cloudflare.com/), where I host my domain. However, [since acme.sh supports many DNS services](https://github.com/Neilpang/acme.sh/tree/master/dnsapi), you can also choose the one you like.

## Installation of acme.sh MM2019:

    $ cd ~
    $ mkdir letsencrypt
    $ cd letsencrypt
    $ wget https://github.com/Neilpang/acme.sh/archive/master.tar.gz

## MM2019:  wget not enabled for SSL, so use curl as follows

    $ curl -k -O -L https://github.com/Neilpang/acme.sh/archive/master.tar.gz
    $ tar xvf master.tar.gz
    $ cd acme.sh-master/
    $ ./acme.sh --install --nocron --home /usr/local/share/acme.sh --accountemail "email@gmailcom"

## Configuring DNS

    $ cd ~/letsencrypt/acme.sh-master/dnsapi
    $ nano dns_cf.sh
    $ uncomment line 4 and line 6 and paste in our API values from CloudFlare into this script and save the file

For CloudFlare, we will set two environment variables that acme.sh 
(specifically, the `dns_cf` script from the `dnsapi` subdirectory) 
will read to set the DNS record. 

You can get your CloudFlare [API key here](https://dash.cloudflare.com/profile).

    export CF_Key="MY_SECRET_KEY_SUCH_SECRET"
    export CF_Email="myemail@example.com"

In case you use another DNS service, check the `dnsapi` directory. Instructions for many DNS providers are already included. You can also find instructions on how to add another DNS service there, although that requires some software development skills.

## Creating the certificate
Now it's time to create the certificate for your domain:

    $ cd /usr/local/share/acme.sh
    $ export CERT_DOMAIN="your-domain.tld"
    $ export CERT_DNS="dns_cf"
    $ ./acme.sh --issue -d "$CERT_DOMAIN" --dns "$CERT_DNS" \
          --cert-file /usr/syno/etc/certificate/system/default/cert.pem \
          --key-file /usr/syno/etc/certificate/system/default/privkey.pem \
          --fullchain-file /usr/syno/etc/certificate/system/default/fullchain.pem \
          --reloadcmd "/usr/syno/sbin/synoservicectl --reload nginx" \
          --dnssleep 20

Please note that this will replace your Synology NAS system default certificate directly.

---------------------------------------------------------------------------------------------------------
## Alternative method that preserves your Synology NAS system default certificate

    $ export CERT_FOLDER="$(find /usr/syno/etc/certificate/_archive/ -maxdepth 1 -mindepth 1 -type d)"
    $ # Make sure $CERT_FOLDER is only one name. Else you have to manually specify the folder.
    $ export CERT_DOMAIN="your-domain.tld"
    $ export CERT_DNS="dns_cf"
    $ ./acme.sh  --issue -d "$CERT_DOMAIN" --dns "$CERT_DNS" \
        --cert-file "$CERT_FOLDER/cert.pem" \
        --key-file "$CERT_FOLDER/privkey.pem" \
        --fullchain-file "$CERT_FOLDER/fullchain.pem" \
        --capath "$CERT_FOLDER/chain.pem" \
        --reloadcmd "/usr/syno/sbin/synoservicectl --reload nginx" \
        --dnssleep 20

Now you can check the DSM control panel - Security - Certificates to see the nominated certificate has been replaced by letsencrypt one. You can now configure to use this one as default and assign to specific services, like vpn, sftp, etc. 
If you see the Lets Encrypt certificate but it's not being used by DMS yet assign the "system default" service to another certificate (create a self signed one if needed) and after the webserver has restarted assign the "system default" service back to the Lets Encrypt certificate. After the webservice has restarted DSM will be using the lets encrypt certificate. 

## Configuring Certificate Renewal
To auto renew the certificates in the future, you need to configure a task in the task scheduler. It is not advised to set this up as a custom cronjob (as was previously described in this wiki page) as the DSM security advisor will tell you that you have a critical warning regarding unknown cronjob(s).

In DSM control panel, open the 'Task Scheduler' and create a new scheduled task for a user-defined script.  

* General Setting: Task - Update default Cert. User - root
* Schedule: Setup a monthly renewal. For example, 11:00 am of the 2nd day every month.
* Task setting: User-defined-script **(modify where needed!)**:

```
# Note: The $CERT_FOLDER must be hardcoded here since the running environment is unknown. Don't blindly copy&paste!
# if you used the normal method the certificate will be installed in the system/default directory
CERTDIR="system/default"
# if you used the alternative method it is copied to an unknown path, change the following example to the output of the creation process and uncomment. 
#CERTDIR="_archive/AsDFgH"

# do not change anything beyond this line!
CERTROOTDIR="/usr/syno/etc/certificate"
PACKAGECERTROOTDIR="/usr/local/etc/certificate"
FULLCERTDIR="$CERTROOTDIR/$CERTDIR"

# renew certificates, this used to be explained as a custom cronjob but works just as well within this script according to the output of the task. 
/usr/local/share/acme.sh/acme.sh --cron --home /usr/local/share/acme.sh/

# find all subdirectories containing cert.pem files
PEMFILES=$(find $CERTROOTDIR -name cert.pem)
if [ ! -z "$PEMFILES" ]; then
        for DIR in $PEMFILES; do
                # replace the certificates, but never the ones in the _archive folders as those are all the unique
                # certificates on the system.
                if [[ $DIR != *"/_archive/"* ]]; then
                        rsync -avh "$FULLCERTDIR/" "$(dirname $DIR)/"
                fi
        done
fi

# reload
/usr/syno/sbin/synoservicectl --reload nginx

# update and restart all installed packages
PEMFILES=$(find $PACKAGECERTROOTDIR -name cert.pem)
if [ ! -z "$PEMFILES" ]; then
	for DIR in $PEMFILES; do
              #active directory has it's own certificate so we do not update that package
              if [[ $DIR != *"/ActiveDirectoryServer/"* ]]; then
		rsync -avh "$FULLCERTDIR/" "$(dirname $DIR)/"
		/usr/syno/bin/synopkg restart $(echo $DIR | awk -F/ '{print $6}')
              fi
	done
fi
```
Now you should be all good. 

--------------------------------------------------------------------------------------------------------------------

## Fix a broken environment after Synology DSM upgrade

    $ cd /usr/local/share/acme.sh
    $ ./acme.sh --upgrade --nocron --home /usr/local/share/acme.sh

or manually add below line into /root/.profile

    . "/usr/local/share/acme.sh/acme.sh.env"
