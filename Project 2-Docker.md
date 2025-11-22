### Install Docker Engine

For this project I will be installing the Docker engine in my Fedora VM which uses the PLASMA DE.
[Docker Install](https://docs.docker.com/engine/install/)

**Uninstall old versions**

 Linux may provide unofficial Docker packages --> uninstall these to avoid conflict with official packages
 
![alt](image/uninstall_old_docker_pkgs.png)

**Install using the rpm repository**

(Install Docker Engine by downloading the RPM package, installing it manually, and managing the upgrades manually)

**Set up the repository**

Run the commands `sudo snf -y install dnf-plugins-core` and `sudo dnf-3 config-manager --add-repo https:download.docker.com/linux/fedora/docker-ce-repo`

**Install Docker Engine**

`sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

This could prompt you to accept the GPG. You have to make sure it matches '060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35'

![alt][./image/OpenPGP_key.png]

**Start Docker Engine**

The command `sudo systemctl enable --now docker` configures the Docker systemd service when you boot your system.

After this, you can verify that your installation was successful by running the hell-world image with `sudo docker run hello-world`.

![alt][./image/docker_installation_success.png]

### Install Docker Compose

For a Linux system, you should run the commands `sudo apt update` and `sudo apt install docker-compose-plugin`. If you're working with a non-debian version of Linux like I am, you may need to use dnf in place of apt. Test that these commands worked with `docker compose version`

### Application Setup

For my application, I decided to install BookStack.
[Bookstack Installation](https://www.bookstackapp.com/docs/admin/installation/) [Bookstack Youtube Guide](https://www.youtube.com/watch?v=VQj5kg7orAM)

On the Installation page for BookStack:
1. Click Docker link on list of hosting options
2. Community docker setups are available on the next page, I chose "GitHub Repository" under the LinuxServer.io section
3. By clicking on this, it takes you to the GitHub page for docker-bookstack where you can find the docker-compose file [GitHub .yml](https://github.com/linuxserver/docker-bookstack)
4. Created a directory for bookstack `mkdir bookstack` -> `cd bookstack`
5. In my Fedora VM, I ran `nano docker-compose.yml` and pasted the docker-compose code it provided on the GitHub
	1. Set database password for bookstack
	2. Change the path to the bookstack /config file under "volumes:" to wherever you want to make bookstack
6. Run `sudo docker compose up -d` (-d makes sure that when you close out your server it doesn't get rid of the container)
![alt][./image/bookstack_container_creation.png]
8. Go to your browser and navigate to the URL you specified in your .yml (I used localhost). This will load the login page for Bookstack.
9. The default username is admin@admin.com with the password **password**. This will take you to your home page. 
![alt][./image/bookstack_homepage.png]

### Problems & Solutions

**Unkown Error**
When trying to load the login page for Bookstack I first was unable to load this page at all and discovered I needed to add a database and specify variables for this database in my .yml file. After this error, I kept getting a page that said 'An unkown error has occurred' and to fix this I ran the command `sudo docker compose logs bookstack` to see if I could find the errors explicitly in the logs. Unfortunately I was not able to for this error, but I found that changing where my config files were changed (adding the '.' before the path `./bookstack/config:/config` / moved the filed from the root to the home directory) and not including @'s in my passwords fixed the error. I believe the issue was I didn't include quotations around my password that included the '@' symbol, and this confused bash.
