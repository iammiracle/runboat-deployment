Video walkthrough: https://youtu.be/1BKVH2HUuig | https://youtu.be/sYCKRo7wJVg

- Create virtual machine `instance` (Ubuntu/2GM Memory)

    Run following Cmds on vitual machine to get latest patches:
    ```
    sudo apt update && sudo apt -y upgrade
    ```
- Install Docker and docker-compose:
  - Install docker: 
    ```
    sudo snap install docker
    sudo groupadd docker
    sudo usermod -aG docker $USER
    sudo reboot
    ```
  - Log back in and install docker compose: 
    ```
    sudo curl -L https://github.com/docker/compose/releases/download/v2.11.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    ```

- Update DNS records. Examples for the controller and odoo builds live endpoint:

    - Controller(A record): `runboat-controller-tmp.{base-domain} instance-IP`
    - Builds(A record): `*.runboat-builds-tmp.{base-domain} instance-IP`

- Create postgres database instance and collect credentials.
  - Example root username and password
    - DB root user: `root`
    - DB root pass: `runboat_runboat`
    - DB hostname:  `runboat.csej1ip8qm8x.us-east-1.rds.amazonaws.com`

- Open Database instance `5432` port and disable external fire wall(if any) to the virtual machine for ease of installation. We should turn this back on after setup is complete

- Update/check firewalls so that the virtual machine can connect to database

- Login into virtual machine and check if the virtual machine can connect to the database
    - Install psql client:
      ```
      sudo apt install postgresql-client-common
      sudo apt install postgresql-client-14
      ```
    - Check database connection:
      ```
      PGPASSWORD='runboat_runboat' psql -h <db-host> -d postgres -U root
      ```
    - Clone the deployment repo: `git clone --recursive https://github.com/strativ-dev/runboat-deployment.git`
    - Change dir into the repo: `cd runboat-deployment`

- Configure Kubernetes using microk8s
  - Run `bash resources/microk8s-setup.sh` script and *exit the ssh session*
  - Login back, change dir into `cd runboat-deployment/resources` and run `bash haproxy-install.sh`

- Configure oca runboat repo and make adjustments
  - Change directory back to `runboat-deployment` and run `microk8s config > runboat/kubeconfig` to create kubeconfig
  - Copy docker-compose.yml: `cp resources/docker-compose.yml runboat/.`
  - Update `docker-compose.yml` inside `runboat/`:
    - put DB connection details(host, port, root username) at `RUNBOAT_BUILD_ENV`.
    - put DB password at `RUNBOAT_BUILD_SECRET_ENV`
    - put Controller url at `RUNBOAT_BASE_URL`. Example: `http://runboat.erp360.strativ.se:8000`
    - put Builds domain at `RUNBOAT_BUILD_DOMAIN`. Example: `runboat-builds-tmp.erp360.strativ.se`
    - create personal github token from `https://github.com/settings/tokens/new` with permissions and put at `RUNBOAT_GITHUB_TOKEN`
    - Put a secret value at *Secret* section and update `RUNBOAT_GITHUB_WEBHOOK_SECRET` at `docker-compose.yml`

    - Run docker-compose: `docker-compose up` inside `runboat/`
    - Visit 8000 port at base url to check if its working. e.g. `http://runboat.erp360.strativ.se:8000`
    - Select the odoo plugin repo and configure `RUNBOAT_REPOS` based on the repo name
    - Configure webhook url in the target repo
      - Put `{controller-base-url}:8000/webhooks/github` at `Payload URL` section. e.g. `http://runboat.erp360.strativ.se:8000/webhooks/github`
      - Set `Content type` to `application/json`
      - Put the value of `RUNBOAT_GITHUB_WEBHOOK_SECRET` at secret section
      - Select `push` && `pull-request` from events and save

- Finally, if everythings done accordingly, make changes to the repo and push to target branch and the visit the deployment url configured. e.g. `http://runboat-controller-tmp.erp360.strativ.se:8000/webui/build.html?name=`
