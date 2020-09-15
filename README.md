# RPi Docker
A basic example of how to get a few Docker containers up and running on a Raspberry Pi. This repo sets up 5 containers: NodeJS, [PiHole](https://github.com/pi-hole/docker-pi-hole), [Netdata](https://github.com/netdata/netdata), [Google OAuth](https://github.com/thomseddon/traefik-forward-auth) and [Traefik](https://github.com/containous/traefik/). The PiHole, Netdata and Traefik web portals will be accessible via Google OAuth.

Google OAuth2 enables you to use your Google account to sign in to your services. Using [Google OAuth](https://github.com/thomseddon/traefik-forward-auth) with [Traefik](https://github.com/containous/traefik/) will allow you to whitelist accounts, implement Google’s 2FA, as well as provide a Single Sign-On (SSO) to your services. This not only offers the convenience of not having sign-in frequently but also improves security.

To make the containers accessible to the outside world you will have to setup port forwarding on your router. You will most likely also want to setup Dynamic DNS and [ddclient](https://github.com/ddclient/ddclient). Heres how to do it with [Google Domains](https://support.google.com/domains/answer/6147083?hl=en).

```bash
rpi-docker
|── app/
|   |── src/
|   |   └── server.js
|   |── public/
|   └── Dockerfile
|── pihole/
|   |── etc-dnsmasq.d/
|   |── etc-pihole/
|   └── logs/
|       └── pihole.log
|── traefik-v1/
|   |── acme.json
|   └── traefik.toml
|── traefik-v2/
|   |── acme.json
|   |── traefik.yml
|   └── logs/
|       |── access.log
|       └── traefik.log
|── docker-compose.t1.yml
└── docker-compose.t2.yml
```

## Docker Install

If you already have Docker and Docker Compose installed skip this step.
```bash
# download the Docker install script
curl -fsSL https://get.docker.com -o get-docker.sh
# run the install script
sudo sh get-docker.sh
# add your user to the docker group *optional
sudo usermod -aG docker ${USER}
# if you added your user to the docker group log out and log back in
# test docker install
docker version
docker run hello-world
# install Docker Compose
# if you do not have python 3 installed run the following
sudo apt-get install libffi-dev libssl-dev
sudo apt install python3-dev
sudo apt-get install -y python3 python3-pip
# install Docker Compose
sudo pip3 install docker-compose
```

## Setup
I'm assuming you already have node and yarn installed on your system.
1. Clone and install dependencies:
    ```bash
    git clone git@github.com:j-Riv/rpi-docker.git
    cd rpi-docker/app
    yarn install
    # build image
    docker build --tag nodejs-image-placeholder:latest .
    ```
2. Create DNS Records for domain and sub-domains, <i>ex: example.com, oauth.example.com, traefik.example.com, netdata.example.com and pihole.example.com</i>.

    Assign a static ip to your Raspberry Pi. Setup Port Forwarding on your Router to your Raspberry Pi (80/443). You might want to also setup Dynamic DNS.
3. Configure Google OAuth
    - [Google Developers Console](https://console.developers.google.com)
        - Google Client ID
        - Google Client Secret
    - Create a Google Project: 
    
        We need to create a Google Project that will contain our Web App, Consent Screen and Credentials. Go to [Google Developers Console](https://console.developers.google.com). If prompted, you'll need to agree to the Google Cloud Platform Terms of Service in order to use their API. It's free to use Google's OAuth service. Click on Select a project and New project.

        Enter a unique name to identify the project, such as “Traefik OAuth”. Click Create.

    - Create Credentials:

        Now that our project has been created, we need to create a client ID and client secret in order to authenticate with Google. Choose our Traefik OAuth project, and under the Navigation menu select APIs & Services > Credentials. Click on Create Credentials > OAuth client ID.

    - Configure the Consent Screen:

        Once you click OAuth Client ID, you will see the note to configure the consent screen as shown below. A configuring a consent screen is required before proceeding.

        If you're not automatically prompted, select the OAuth consent screen from the left panel.

        Choose a name for your app, such as "Traefik OAuth", then under the Authorized domains section enter your domain, for instance, “example.com”. Make sure that you press Enter to add it, and then click Save.

        After hitting Save, you will return back to creating an OAuth Client ID.
    
    - Create the OAuth client ID

        Now select the Web Application type and enter a name for your web application, such as “Traefik”. In addition, you'll need to enter your Authorized redirect URI as https://oauth.example.com/_oauth. Make sure that you press Enter to add it, and then click Save.

        Note: You are only allowed to add redirect URIs that direct to your Authorized Domains. Return to the OAuth consent screen if you need to edit them.

        The credentials for our SSO for Docker have been created! Copy and save the client ID and client secret; we'll need to setup the Docker Compose files later.

        Generate a random secret with: `openssl rand -hex 16` this will be used to sign the Google OAuth cookie. Copy and save this random secret as well.

4. Docker Container Names Resolution:
    If you want to have your container names resolved by netdata it needs to have access to docker group. To achive that just add environment variable PGID=999 to netdata container, where 999 is a docker group id from your host. This number can be found by running:
    ```bash
    grep docker /etc/group | cut -d ':' -f 3
    ```
5. Create .env and add values:
    ```bash
    DOMAIN_NAME={YOUR DOMAIN} # main domain
    # GOOGLE OAUTH
    GOOGLE_CLIENT_ID={GOOGLE CLIENT ID} # Google Client ID from step 3
    GOOGLE_CLIENT_SECRET={GOOGLE CLIENT SECRET} # Google Client Secret from step 3
    OAUTH_SECRET={RANDOM STRING} # random secret from step 3
    EMAIL_WHITELIST={COMMA SEPERATED EMAIL LIST FOR GOOGLE OAUTH} # ex: name@email.com,anothername@email.com
    # PiHOLE
    SERVER_IP={LOCAL IP} # local ip
    PIHOLE_WEBPASSWORD={RANDOM STRING} # the password to be used for PiHole Web Portal
    # NetData Docker Container Names Resolution
    DOCKER_PID={PROCESS ID} # from step 4
    ```
6. Traefik Config:
    - Traefik v1 - <i>/traefik-v1/</i>
        - traefik.toml
            - Replace email, domain and sub-domains
    - Traefik v2  - <i>/traefik-v2/</i>
        - traefik.yml
            - Replace email

    - Create acme.json in appropriate directory
        ```bash
        # v1
        touch /traefik-v1/acme.json
        chmod 600 /traefik-v1/acme.json
        #v2
        touch /traefik-v2/acme.json
        chmod 600 /traefik-v2/acme.json
        touch /traefik-v2/logs/access.log /traefik-v2/logs/traefik.log
        ```
7. Run
    ```bash
    # Traefik v1
    docker-compose -f docker-compose-t1.yml up -d
    # Traefik v2
    docker-compose -f docker-compose-t2.yml up -d
    ```



