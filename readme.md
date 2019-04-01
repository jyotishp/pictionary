# Pictionary

# App structure
There are two parts to the app.
- One dealing with the game assignments, recording game sessions, authentication, etc (Django)
- One efficiently implementing the realtime chat and drawing canvas updates (NodeJS + SocketIO)

Both the instances can be proxied using Apache or Nginx. The advantage with this setup is that we can fastrack the development part of the game assignments and authentication part by using a mature and easy to implement framework like Django and leave the websockets to NodeJS which it is good at.

The Django app rests in `pictionary` directory and Nodejs + SocketIO app is in `pictionary-sockets`.

# Setup instructions
## Preparing the Django App
### Install Dependencies
It is recommended to create a virtual environment to keep things simple on your end. You can create a virtual environment by using

```bash
virtualenv -p python3 --no-site-packages <path_to_virtual_env>
```

Replace the `<path_to_virtual_env>` with the path where you want to install yout environment. In case you create the environment in the root directory, the files should be ignored by `git`. If they are not, please add them to the `.gitignore` file.

Now, activate and install the dependencies.

```bash
source <path_to_virtual_env>/bin/activate
pip install -r <root_of_repo>/pictionary/requirements.txt
```

### Prepare the environment
Have a look at `<root_of_repo>/pictionary/.env.example` file. You need to replace the values with the ones you're working with. Once you have the values updated, you need to load them to your environment.

*You can also have them declared in your virtual env to avoid loading them seperately.* **NEVER PUSH THE KEYS TO THE REPO.**

Once the environment variables are loaded, complete any pending migrations.

```bash
cd <root_of_repo>/pictionary
python manage.py makemigrations
python manage.py migrate
```

Now, that everything is setup, you can run the server. During development, it's okay to run Django's in built web server but it's gunicorn (or equivalent) is recommended for production deployments. On the same note, sqlite3 should not be used in production.

### Starting the app
```bash
python manage.py runserver
```

## Preparing the NodeJS App
### Install Dependencies
Install NodeJS and run the following

```bash
npm install
```

### Starting the app
```bash
node index.js
```

## Preparing the proxy
I use Nginx but the configuration for Apache HTTPD should more or less be the same. You can look into `ProxyPass` and `ProxyPassReverse` directives for HTTPD.

First, install Nginx (refer your distribution's package manager for the same).

Update your Nginx config to include the following.

```
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	return 301 https://pictionary.mrdx.ml$request_uri;
}

server {
	listen 443 ssl default_server;
	listen [::]:443 ssl default_server;
	ssl_certificate     /etc/pki/pictionary.mrdx.ml/fullchain.pem;
	ssl_certificate_key /etc/pki/pictionary.mrdx.ml/privkey.pem;
	ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers         HIGH:!aNULL:!MD5;

	server_name pictionary.mrdx.ml;
	proxy_set_header X-Forwarded-Proto $scheme;
	proxy_set_header X-Forwarded-Host $host;

	location ~ ^/socket.io {
	 	proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
		proxy_pass http://127.0.0.1:3000;
	}

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		proxy_pass http://127.0.0.1:8000;
		#try_files $uri $uri/ =404;
	}
}
```

You can ignore `default_server` if your are running multiple domains on your instance. Replace `pictionary.mrdx.ml` with whatever your domain name is. If you don't have a domain name (which facebook's OAuth requires), you can always hardcode it in your `/etc/hosts`. You can also avoid HTTPS if you don't want to have it.
