# Set up Fail2Ban on Ubuntu server
Fail2Ban protects your services from brute-force attacks by monitoring login failures in your app logs and automatically blocking suspicious IPs with firewall rules. It’s customizable to your log formats, works well with Docker and reverse proxies, and enforces temporary bans to prevent unauthorized access without locking out legitimate users permanently. This helps keep your authentication endpoints secure and reduces manual intervention.

### 1. Install Fail2Ban
```
apt install fail2ban
systemctl enable fail2ban
systemctl start fail2ban
```

### 2. Configure your rules
A rule means the thing you want to block, here we want to block the ip addresses that try to connect to the owncloud and flask service.

In the file */etc/fail2ban/jail.d/flask-login.local*, configure your rules.

- Example :

```
[DEFAULT]
allowipv6 = auto
loglevel = INFO
logtarget = /var/log/fail2ban.log

[sshd]
enabled = true
port = ssh
filter = sshd
backend = systemd
maxretry = 5
bantime = 600
findtime = 600

[owncloud-login]
enabled = true
filter = owncloud-login
action = iptables-docker[name=FlaskLogin, port=http, protocol=tcp]
logpath = /var/lib/docker/volumes/myowncloud_owncloud_data/_data/files/owncloud.log
bantime = 3600
findtime = 600
maxretry = 5


[flask-login]
enabled = true
filter = flask-login
action = iptables-docker[name=FlaskLogin, port=http, protocol=tcp]
logpath = /home/babtou/MyOwnCloud/logs/flask.log
bantime = 3600
findtime = 600
maxretry = 5
```
In "action" we have "**iptables-docker**", we will remember this name (see the **4.**)...

The rule **SSHD** is obligatory !
You have **to mount volumes** for each services to find the file path logs.

- Example *docker-compose.yml* for owncloud service :
```
volumes:
      - owncloud_data:/mnt/data
```
The file path log is here : ***/var/lib/docker/volumes/myowncloud_owncloud_data/_data/files/owncloud.log***.

### 3. For each rules, create file *.conf*
For each rules, create the file : */etc/fail2ban/filter.d/**<nam_rule>**.conf*
Inject your configuration.

- Example rule *flask-login* in the file *flask-login.conf*: 
```
[Definition]
failregex = "message":"Login failed: '.*' \(Remote IP: '<HOST>'\)"
ignoreregex =
```
The regex depends on the message is generated when (in our case) a user is trying to connect in the auth page but his ids are bad...

- Example *flaskapp* log, login failed for user : 
```
2025-05-26 11:57:27,050 WARNING: Login failed for user 'aaa' [IP: 192.168.32.6]
```

If these logs are not generating, you have to custom your ***app.py***...

- Example *./flaskapp/**app.py*** for admin auth : 

```
import logging
from werkzeug.middleware.proxy_fix import ProxyFix


app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1)

logger = logging.getLogger(__name__)
handler = logging.FileHandler('<<the path of your log>>')
formatter = logging.Formatter('%(asctime)s %(levelname)s: %(message)s [IP: %(ip)s]')
handler.setFormatter(formatter)
logger.setLevel(logging.INFO)

if not logger.handlers:
    logger.addHandler(handler)

@app.route("/admin-login", methods=["GET", "POST"])
def admin_login():
    if redirect_response:
        return redirect_response

    if request.method == "POST":
        ip = request.remote_addr
        username = form.username.data  # ou autre champ selon ton formulaire
        logger.warning(f"Login failed for user '{username}'", extra={"ip": ip})

    return render_template('login.html', form=form, alert=alert, ip_dashboard=owncloud_page())
```
### 4. Configure your "access-list"
Copy the file : */etc/fail2ban/action.d/iptables.conf* to this file : */etc/fail2ban/action.d/iptables-docker.conf*
In the file you created (here ***iptables-docker.conf***, see the **2.**), go in the part **[Init]** and **[Definition]**, to add your settings.

- Lines to change :
```
[Init]
chain = DOCKER-USER
# port =
# protocol = 

[Definition]
actionstart = <iptables> -N f2b-<name> || true
              <iptables> -I <chain> -j f2b-<name>
              <iptables> -F f2b-<name>
```


### 5. Fail2Ban commands
- Start/Stop/Restart Fail2Ban : 
```
systemctl start fail2ban
systemctl stop fail2ban
systemctl restart fail2ban
```
- See **owncloud-login** rule logs (number of fails, IP bans, etc.) : 
```
fail2ban-client status owncloud-login
```
- See the rules : 
```
iptables -L DOCKER-USER -n --line-numbers
```
You have to see something like that : 
```
Chain DOCKER-USER (1 references)
num  target     prot opt source               destination
1    f2b-FlaskLogin  0    --  0.0.0.0/0            0.0.0.0/0
```
- See IPs ban : 
```
iptables -L f2b-FlaskLogin -n
```
Or : 
```
fail2ban-client status owncloud-login
```
- Unban IP :
```
fail2ban-client set flask-login unbanip 192.168.1.25
```
- Delete a rule (here the 2nd rule) : 
```
iptables -D DOCKER-USER 2
```
