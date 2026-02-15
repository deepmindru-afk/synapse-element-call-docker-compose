# Generating the necessary config files
In order to run the container, config files for Homeserver need to be generated:
```bash
sudo docker run -it --rm --mount type=bind,src=./data,dst=/data -e SYNAPSE_SERVER_NAME=domain.tld -e SYNAPSE_REPORT_STATS=no matrixdotorg/synapse:latest generate
```
This Docker command does the following:
- Generate the `homeserver.yml` config file under `data/`
- Sets the domain to be used for the Homeserver
- Deletes the container used for generated after completion

# Create and run the Docker Compose file
## Synapse and Synapse database
```yaml
  synapse:
    image: docker.io/matrixdotorg/synapse:latest
    restart: always
    environment:
      - SYNAPSE_CONFIG_PATH=/data/homeserver.yaml
    volumes:
      - ./data:/data
    depends_on:
      - db
    ports:
      - 8448:8448/tcp

  db:
    image: docker.io/postgres:15-alpine
    environment:
      - POSTGRES_USER=synapse
      - POSTGRES_PASSWORD=CHANGE_ME_0
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --lc-collate=C --lc-ctype=C
    volumes:
      - ./db:/var/lib/postgresql/data
```
The Synapse port 8448 does not need to be served at the root of `domain.tld` but can be served at the root of `matrix.domain.tld`.

The environmental variable `POSTGRES_PASSWORD` (`CHANGE_ME_0`) needs to be changed to be the same as the one in the generated `data/homeserver.yml`.

This basic compose file will be enough to just run Matrix with just chat capabilities.

## Element call (Livekit)
```yaml
  auth-service:
    image: ghcr.io/element-hq/lk-jwt-service:latest
    container_name: element-call-jwt
    hostname: auth-server
    environment:
      - LIVEKIT_JWT_PORT=8080
      - LIVEKIT_URL=https://matrix-rtc.domain.tld/livekit/sfu 
      - LIVEKIT_KEY=devkey
      - LIVEKIT_SECRET=CHANGE_ME_1
      - LIVEKIT_FULL_ACCESS_HOMESERVERS=domain.tld
    restart: always
    ports:
      - 8080:8080

  livekit:
    image: livekit/livekit-server:latest
    container_name: element-call-livekit
    command: --config /etc/livekit.yaml
    ports:
      - 7880:7880/tcp
      - 7881:7881/tcp
      - 7882:7882/udp
    restart: always
    volumes:
      - ./livekit/config.yaml:/etc/livekit.yaml:ro

  well-known-server:
    image: nginx:alpine
    volumes:
      - ./well-known:/usr/share/nginx/html/.well-known:ro
      - ./nginx-conf:/etc/nginx
    network_mode: "host"
```
The service for voice and video is provided by Livekit which then in turn relies on the `lk-jwt-service` service for authentication.

The environmental variable `LIVEKIT_SECRET`'s value (`CHANGE_ME_1`) needs to be same with the one defined in `livekit/config.yml` under `keys` -> `devkey`.

The environmental variable `LIVEKIT_FULL_ACCESS_HOMESERVERS` defines the users under which domain have access to use the voice and video service. This is the part of the username after `@user:` (which is usually `domain.tld`).

### Managing well knowns
This step is important as it tells Matrix clients and servers where to find key resources like the Synapse server (using `.well-known/matrix/server`) and WebRTC components (`.well-known/matrix/clients`).

It is important to note that if your `SERVER_NAME` is the base domain (`domain.tld`), your well knowns should be under the same base domain, which in this case would be `domain.tld/.well-known/matrix/client` and `domain.tld/.well-known/matrix/server`.

If your nominated `SERVER_NAME` is under a subdomain, use the subdomain to serve the well knowns (`sub.domain.tld/.well-known/matrix/client`).

In general, the Nginx configuration would need to be serving JSON and CORS being set correctly:
```yaml
    server {
        listen 80;

        location /.well-known/ {
            add_header Access-Control-Allow-Origin "*" always;
            default_type application/json;

            root /usr/share/nginx/html;
            index index.html;
        }
    }
``` 
For this setup, the Nginx container is set to host mode netowrking purely for convenience but it could also just expose ports 80 and 81 using:
```yaml
    ports:
      - 80:80
      - 81:81
```

## Full compose file
```yaml
services:
  synapse:
    image: docker.io/matrixdotorg/synapse:latest
    restart: always
    environment:
      - SYNAPSE_CONFIG_PATH=/data/homeserver.yaml
    volumes:
      - ./data:/data
    depends_on:
      - db
    ports:
      - 8448:8448/tcp

  db:
    image: docker.io/postgres:15-alpine
    environment:
      - POSTGRES_USER=synapse
      - POSTGRES_PASSWORD=CHANGE_ME_0
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --lc-collate=C --lc-ctype=C
    volumes:
      - ./db:/var/lib/postgresql/data

  auth-service:
    image: ghcr.io/element-hq/lk-jwt-service:latest
    container_name: element-call-jwt
    hostname: auth-server
    environment:
      - LIVEKIT_JWT_PORT=8080
      - LIVEKIT_URL=https://matrix-rtc.domain.tld/livekit/sfu 
      - LIVEKIT_KEY=devkey
      - LIVEKIT_SECRET=CHANGE_ME_1
      - LIVEKIT_FULL_ACCESS_HOMESERVERS=domain.tld
    restart: always
    ports:
      - 8080:8080

  livekit:
    image: livekit/livekit-server:latest
    container_name: element-call-livekit
    command: --config /etc/livekit.yaml
    ports:
      - 7880:7880/tcp
      - 7881:7881/tcp
      - 7882:7882/udp
    restart: always
    volumes:
      - ./livekit/config.yaml:/etc/livekit.yaml:ro

  well-known-server:
    image: nginx:alpine
    volumes:
      - ./well-known:/usr/share/nginx/html/.well-known:ro
      - ./nginx-conf:/etc/nginx
    network_mode: "host"
    #ports:
    #  - 80:80
    #  - 81:81
```
Running the compose file
```bash
sudo docker compose up -d
```

## Important ports
The following ports will be exposed to the reverse proxy which in turn exposes these ports to the internet:
1. `8448/tcp` (Synapse - matrix.domain.tld)
2. `8080/tcp` (JWT)
3. `7881/tcp` (Livekit TCP - fallback)
4. `7882/udp` (Livekit UDP - multiplexed)
5. `80/tcp` (Webserver - domain.tld)\*
6. `81/tcp` (Webserver - matrix-rtc.domain.tld)

\*If `domain.tld` already hosts something else, you can just add the well knowns to that webserver and skip this altogether. 

# Initial Matrix homeserver setup
## Creating the admin account and getting the admin token
After the containers have been created, we have to create an initial admin user that we can use:
```bash
sudo docker compose exec -it synapse register_new_matrix_user https://matrix.domain.tld -c /data/homeserver.yaml -a -u YOUR_USERNAME
```
This will prompt you to enter a password for the username you added under `-u YOUR_USERNAME`. Do note that the full username will be `@YOUR_USERNAME:domain.tld`. You will now be able to login to your Homeserver using Element with `domain.tld` as the Homeserver. 

Go to `Settings > Help & About > Advanced` and copy your access token. This will server as your admin token for use in authenticated API calls.

## Generate a registration token
To have other users be able to register for an account, you will have to have a registration token they can use to complete the account setup:
```bash
curl --request POST --url "https://matrix.domain.tld/_synapse/admin/v1/registration_tokens/new" -H "Authorization: Bearer syt_emF5_msYxJbbaNtWqDWOhZWzB_2hFPeK" -d '{"token":"YOUR_REGISTRATION_TOKEN"}'
```
Here, `YOUR_REGISTRATION_TOKEN` is the registration token that the users will have to provide in order to complete their account setup. By default, this will have unlimited uses.

## Setting up email verification
As an added layer of security, email verification is turned on by default. This means you will have to provide the SMTP details of your mailserver and email account you'll use for Matrix.
```yaml
email:
  smtp_host: mail.domain.tld
  smtp_port: 465
  smtp_user: matrix@domain.tld
  smtp_pass: CHANGE_ME_7
  force_tls: true
  require_transport_security: true
  enable_tls: true
  tlsname: mail.domain.tld
  notif_from: Matrix <matrix@domain.tld>
  app_name: Matrix
  enable_notifs: true
  notif_for_new_users: true
  client_base_url: https://matrix.domain.tld/riot
  validation_token_lifetime: 15m
  invite_client_location: https://matrix.domain.tld
  subjects:
    message_from_person_in_room: '[%(app)s] You have a message on %(app)s from %(person)s
      in the %(room)s room...'
    message_from_person: '[%(app)s] You have a message on %(app)s from %(person)s...'
    messages_from_person: '[%(app)s] You have messages on %(app)s from %(person)s...'
    messages_in_room: '[%(app)s] You have messages on %(app)s in the %(room)s room...'
    messages_in_room_and_others: '[%(app)s] You have messages on %(app)s in the %(room)s
      room and others...'
    messages_from_person_and_others: '[%(app)s] You have messages on %(app)s from
      %(person)s and others...'
    invite_from_person_to_room: '[%(app)s] %(person)s has invited you to join the
      %(room)s room on %(app)s...'
    invite_from_person: '[%(app)s] %(person)s has invited you to chat on %(app)s...'
    password_reset: '[%(server_name)s] Password reset'
    email_validation: '[%(server_name)s] Validate your email'
```
Make sure to change the basic SMTP settings to your mailserver and email account as well as some domain-dependent variables:
- `smtp_host`
- `smtp_port`
- `smtp_user`
- `smtp_pass`
- `tlsname`
- `notif_from`
- `invite_client_location`
- `client_base_url`