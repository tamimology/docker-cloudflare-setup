# The A-Z Cloudflare Tunnel Setup in Docker container Guide

#### **NOTE:** *WHEN WRITING THIS GUIDE, IT WAS BASED ON*
- [DOCKER-COMPOSE v2.11.2](https://github.com/docker/compose)
- [CLOUDFLARE TUNNEL v2022.10.2](https://github.com/cloudflare/cloudflared)
- [AUTHELIA v4.36.9](https://github.com/authelia/authelia)
- [DDNS UPDATER v2.4.1](https://github.com/qdm12/ddns-updater)

# Table of Contents

1. <a href="#signing-up-for-a-cloudflare-service">Signing Up for a Cloudflare Service</a>
2. <a href="#creating-a-dns">Creating a DNS</a>
3. <a href="#changing-nameservers">Changing Nameservers</a>
4. <a href="#adding-tunnels">Adding Tunnels</a>
5. <a href="#getting-the-tunnel-token">Getting The Tunnel Token</a>
6. <a href="#creating-an-api-token">Creating an API Token</a>
7. <a href="#updating-your-public-ip">Updating Your Public IP</a>
8. <a href="#the-consolidated-docker-compose">The Consolidated docker-compose</a>


## Signing Up for a Cloudflare Service

#### The first step in the whole setup, is to have a Cloudflare account

Simply, open the official website from [here](https://dash.cloudflare.com/sign-up) to sign up. Just use your preferred email and password.

After that, the main dashboard will open, allowing you to add a new site, which you have already signed up for (check [here](/README.md#creating-a-dns) to see how to get a free DNS service).

#### **NOTE:** *Never use the DuckDNS service with Cloudflare as it is not supported*

![add site](/cloudflare/add_site.png)

After that, it should open your website dashboard by default, if not, simply click on it under the Home view.

At the very bottom right side, where it says ***API***, take note of the **Zone ID** and save it somewhere for later usage.

Now on the right side, click **DNS Settings**.

![dns settings](/cloudflare/dns_settings.png)

In there, click on **Add record** and fill in as following:
  1. **A**
  2. **@**
  3. ***your public IP address***, which you can get it easily from [here](https://www.myip.com/). We will see later on how to make it update automatically
  
  Leave the rest untouched and click on **Save**
  
![dns manager](/cloudflare/dns_manager.png)


Now scroll down till you see the part where it says **Cloudflare Nameservers** and take note of the 2 URLs there for later use.

![ns](/cloudflare/ns.png)


## Creating a DNS

#### Next, is to have a DNS service ready, either a paid one, or choose a free DNS service provider.

In this guide, I will go with the free DNS service from [Freenom](https://www.freenom.com/) as this is what I am using in my home setup. Other DNS providers shall have similar setups which I can not cover them all in here.

Once you enter to the website, you need to check the availability of the desired URL, until you find a free one, or decide to buy a cheap one. In the guide, I will be referring to "***mydomain.gk***" for illustration purposes only

At the time I prepared this guide, there was a bug on their server in which you cannot just click on any of the options available, so to register, simply click on **Services** then choose **Register a New Domain**. Follow the steps in there, and make sure you choose the **12 months** plan as it is the longest period offered for free.

![free dns](/cloudflare/free_dns.png)

## Changing Nameservers

#### This is where you link your DNS address with the Cloudflare Tunnel

Now after you signed up, go to **Services** then choose **My Domains** from the drop down list, then next to your registered domain, click on **Manage Domain**.

In the next window, click on **Management Tools** then choose **Nameservers** and add the *nameservers* you took note of from the previous step

![changing_ns](/cloudflare/changing_ns.png)


## Adding Tunnels

#### In Cloudflare, a Tunnel is similar to a route of the main domain, or simply, think of it as the subdomains.

To create new Tunnel, go to the [**Cloudflare Zero Trust**](https://dash.teams.cloudflare.com/) dashboard, and under **Access**, click on **Tunnels**

Click on **Create a tunnel**, enter a name for that tunnel, i.e. "***My Domain***"

Now the Tunnel is created, and a new page opens showing the **Install connector** environment options available for that created tunnel. Click on "**Docker**" add take note of what is in there for later use.

Click on **Next**

![tunnel docker](/cloudflare/tunnel_docker.png)


Now, here you will be having the option to start adding your subdomains and redirect them to your internal network IP address and port.

Add all your subdomains before leaving this page. **DON'T CREATE SEPERATE TUNNELS FOR EACH SUBDOMAIN, ALL CAN GO IN ONE TUNNEL**

###### To test your subdomain, simply click on it after it has been created and it should redirect you correctly. If not, make sure you followed this guide from the beginning and entered your host IP and port correctly, as I guarantee you 100% it works if you follow me as mentioned.



## Getting The Tunnel Token

#### This token will be needed for usage in the docker-compose file in the last step in this guide

If you recall, in the previous step, a docker command line was created, and I mentioned to take note of it. Well, if you read it, you will see it looks like this

`docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token ` **`ed324rtgdwMTI5Njk5YTI1ODExTHIS ISASAMPLETOKENANDNOTTOBEUSEDATALLIiwicyI6Ik56WmtOVGd5TjJVdE56TTJaaTAwTewrrfgre45tyg4FGGHREAEU0WXpneiJ9`**

You just need to take the highlighted string at the very end after ***`--token `*** and save it somewhere for the next step. *Make sure you don't include the "***SPACE***" at the beginning of the token string*


## Creating an API Token

#### This API token will be needed to have the public IP auto updated in the container that you will create next.

In **Cloudflare** home dashboard, under the **API** section, click on "***Get you API token***", or simply click [here](https://dash.cloudflare.com/profile/api-tokens)

Click on "**Create Token**", and from the very end, choose "***Custom token***" and do as in below

![api token](/cloudflare/api_token.png)

You shall get something similar to the below. Take note of that token and save it somewhere as it will ***NEVER*** be shown again

![token](/cloudflare/token.png)


## Updating Your Public IP

#### If you recall, in Cloudflare dashboard, your public IP was entered in the first step where you added a record under your DNS management part. This might change in various conditions, and need to be up-to-date after the change so you can access your domain remotly

The two main reasons for the public IP to be changed are:
  - A restart of the main router connected to the ISP cable
  - Your ISP decides to continuously change it as you did not pay extra to have a static public IP

Well, in either case, you need to have that IP updated. Rather than everytime checking your new public IP, then going to the Cloudflare's dashboard and updating it manually, there is an automatic way to do that.

You will be using a docker container that checks your IP every 5 minutes, and if it is changed, it will update it with the new vaule, otherwise, it will sleep for another 5 minutes.

But first, you need to create the configuration folder and file for that container. *By the way, this container support various other DNS service providers, in which you can add them in the configuraiton file by following their guide on the container's github page*. I will be only covering the Cloudflare part here.

Now, open your docker "***$PERSIST***" folder and create a folder named "**ddns-updater**". In that folder, create two files. One to stay as *empty*, and is named as "***updates.json***", and another named "**config.json**", or simply download the sample file from [here](/config.json)

In the "*config.json*" file, add the below lines

```json
{
  "settings": [
    {
      "provider": "cloudflare",
      "zone_identifier": "ZONE_ID",
      "domain": "MYDOMAIN.GK",
      "host": "@",
      "ttl": 600,
      "proxied": true,
      "token": "API_TOKEN",
      "ip_version": "ipv4"
    }
  ]
}
```

Make sure to update the following to match yours:
  - ***ZONE_ID*** which you took note in first steps
  - ***MYDOMAIN.GK*** to match you own website address you added in the first step at the Cloudflare's home dashboard
  - ***API_TOKEN*** which you have generated in the previous step

Save the file. Now, for some reason, you need to change the permissions of the created folder and its contents so the container will be able to have write access once created.

Considering that "***UID***=1000" and "***GID***=100", SSH into your host machine in which you will create the contianer and exceute the following commands

`chown -R UID:GID /docker/ddns-updater`


## The Consolidated docker-compose

#### This part is where we start off with creating the Cloudflare Tunnel docker-compose file, along with the ddns updater container to have the public IP auto-updated in case of the ISP changed it suddenly for some reason

You can simply download the sample file from [here](/docker-compose.yml), or manually create one as below

Make sure you upodate the following to match your setup:
  - **$TUNNEL_TOKEN** that you have taken note from the docker command after you created your tunnel above
  - **$PERSIST** to match your docker folder, i.e. "*/some_folder/docker*"
  - **$TZ** to be your local time zone, check [here](http://en.wikipedia.org/wiki/List_of_tz_database_time_zones) for details
  - **PUID** to match you docker user id, i.e. "*1000*"
  - **PGID** to match you docker group id, i.e. "*100*"
  - **$PUSHOVER_API** and **@$PUSHOVER_USER_KEY**, or change the whole *SHOUTRRR_ADDRESSES* if you use something else

```yaml
#
####################################################
#                                                  #
#              -------CloudFlare-------            #
#                                                  #
####################################################
#
  cloudflare:
    container_name: cloudflare
    restart: always
    user: root
    environment:
      - NO_AUTOUPDATE="true"
      - TUNNEL_TOKEN=$TUNNEL_TOKEN
    networks:
      bridge:
         ipv4_address: 172.19.0.201
    command: 'tunnel --config /etc/tunnel/config.yml run'
    image: 'cloudflare/cloudflared:latest'
#
####################################################
#                                                  #
#             -------DDNS-Updater-------           #
#                                                  #
####################################################
#
  ddns-updater:
    container_name: ddns-updater
    restart: always
    environment:
      - TZ=$TZ
      - PUID=$PUID
      - PGID=$PGID
      - PERIOD=5m
      - UPDATE_COOLDOWN_PERIOD=5m
      - PUBLICIP_FETCHERS=all
      - PUBLICIP_HTTP_PROVIDERS=all
      - PUBLICIPV4_HTTP_PROVIDERS=all
      - PUBLICIPV6_HTTP_PROVIDERS=all
      - PUBLICIP_DNS_PROVIDERS=all
      - PUBLICIP_DNS_TIMEOUT=3s
      - HTTP_TIMEOUT=10s
      - LISTENING_PORT=8000
      - ROOT_URL=/
      - BACKUP_PERIOD=24h # 0 to disable
      - BACKUP_DIRECTORY=/updater/data
      - LOG_LEVEL=info
      - LOG_CALLER=hidden
      - SHOUTRRR_ADDRESSES=pushover://shoutrrr:$PUSHOVER_API@$PUSHOVER_USER_KEY
    volumes:
      - $PERSIST/ddns-updater:/updater/data
    networks:
      bridge:
    ports:
      - 8002:8000/tcp
    image: 'qmcgaw/ddns-updater:latest'
#
```
Now if you try to access your local ip address with port 8002, you should get something like this

![ddns](/cloudflare/ddns.png)




***NOTE:*** *if you want to have access to Cloudflare via **Authelia**, you can refer my guide from [here](https://github.com/tamimology/cloudflare-authelia)*






## License
This document guide is licensed under the CC0 1.0 Universal license. The terms of the license are detailed in [LICENSE](/LICENSE)
