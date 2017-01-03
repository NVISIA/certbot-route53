# certbot-route53
This docker image is used to automatically request or renew Let's Encrypt signed SSL keys for your domain(s) that use Amazon's Route53 DNS service.

[Docker Hub link](https://hub.docker.com/r/ntcnvisia/certbot-route53/)

### Example
```
docker run -e "DOMAIN=[YOUR DOMAIN]" -e "EMAIL=[YOUR EMAIL]" -e "AWS_ACCESS_KEY_ID=[YOUR AWS KEY]" -e "AWS_SECRET_ACCESS_KEY=[YOUR AWS SECRET KEY]" -e "AWS_DEFAULT_REGION=[YOUR AWS REGION]" -e "TZPATH=America/Chicago" -v certbot-route53-letsencrypt:/etc/letsencrypt  ntcnvisia/certbot-route53
```

### Required Volumes
You should create a named volume (`volume create --name [YOUR VOLUME NAME]`).
This volume will be mapped to /etc/letsencrypt - the folder certbot uses to store certificates and other configuration data required for renewal.

### Runtime Configuration
This image is configured using the following environment variables:

Variable Name | Purpose
------------- | -------
DOMAIN | Comma separated list of domains to request a single certificate for.
EMAIL | Administrator email for Let's Encrypt recovery purposes.
TZPATH | Name of the timezone to use for the container. This must match the host for AWS request signing to work! This must be a standard name ie: America/Chicago as it's used as a part of the /usr/share/zoneinfo/ path.
FORCERENEWAL | If this variable is defined, the [--force-renewal flag][re-run-certbot] will be applied to certbot. This forces a certificate update.
EXPAND | If this variable is defined, the [--expand flag][re-run-certbot] will be applied to certbot. This allows SAN names to be added to an existing certificate.
AWS_ACCESS_KEY_ID | AWS-provided access key. Must have permissions for Route53 in the correct zone. See policy information below
AWS_SECRET_ACCESS_KEY | AWS-provided secret key. Must have permissions for Route53 in the correct zone. See policy information below
AWS_DEFAULT_REGION | AWS Region to use for Route53 access.
STAGING | If this variable is set at all, hit the Let's Encrypt Staging environment instead of the real one. Only use this for testing, as the certificates will not be valid.

### AWS Policy Information
You will want to create a separate AWS account and policy with limited permissions for Route53 where the hosted zonefile is limited to the subdomains you want certificates for.

Example policy:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets",
                "route53:GetChange",
                "route53:ListResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/[YOUR ZONEFILE ID HERE]"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:GetChange"
            ],
            "Resource": [
                "arn:aws:route53:::change/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListHealthChecks",
                "route53:ListHostedZones",
                "route53:ListHostedZonesByName"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

### Extension
While this image works out of the box for extremely simple workflows, it can also be used as a base image to automate the install process for your other containers.
To do so, simply create an image using this image as a base, and override one of the three files mentioned below.
Images based on this image will still need to provide the environment variables mentioned above, and scripts may therefore use them.

File to Override | Purpose
---------------- | -------
/root/certbot-route53/hook-pre.sh | Called before any domains are renewed, useful for stopping things that must be terminated before it's safe to change certificates
/root/certbot-route53/hook-each.sh | Called once for each domain that is successfully renewed. Receives two additional environment variables: `$RENEWED_LINEAGE` which points to the live subdirectory in the provided volume for the domain in question, and `$RENEWED_DOMAINS` which has a space-delimited list of domains that were renewed
/root/certbot-route53/hook-post.sh | Called after all domains are renewed, useful for starting things up again or renaming and copying the full set of files to a different volume

An example of this will be provided soon™

[re-run-certbot]:https://certbot.eff.org/docs/using.html#re-running-certbot
