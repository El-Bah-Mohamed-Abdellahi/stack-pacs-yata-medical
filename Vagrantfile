# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
   
  config.vm.box = "fedora-base-server"
   
  config.vm.hostname = "pacs-yata-server"
  config.vm.network "private_network", ip: "192.168.56.10"

   # --- Réseau ---

  config.vm.network "forwarded_port", guest: 8080, host: 8080   # Admin dcm4chee
  config.vm.network "forwarded_port", guest: 8083, host: 8083   # Viewer MedDream
  config.vm.network "forwarded_port", guest: 11112, host: 11112 # Port DICOM
  config.vm.network "forwarded_port", guest: 8081, host: 8181   # Token service
  config.vm.network "forwarded_port", guest: 8082, host: 8082   # Viewer integration
   # --- Ressources ---
  config.vm.provider "virtualbox" do |vb|
    vb.name = "pacs-yata-server"
    vb.memory = "6144"
    vb.cpus = 4
  end
  config.vm.provision "shell", name: "fix-network", run: "always", inline: <<-SHELL
    set -euo pipefail

    echo ">>> Suppression de la route par défaut indésirable sur enp0s8 (réseau privé)"
    ip route del default via 192.168.56.1 dev enp0s8 2>/dev/null || true

    echo ">>> Route par défaut finale"
    ip route show default
  SHELL

   # 1. Installation de Docker sur Fedora
  config.vm.provision "shell", name: "install-docker", inline: <<-SHELL
    set -euo pipefail

    echo ">>> Mise à jour du système"
    dnf -y update

    echo ">>> Prérequis"
    dnf -y install dnf-plugins-core curl

    echo ">>> Dépôt Docker officiel"
    dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo

    echo ">>> Installation Docker"
    dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    echo ">>> Activation du service"
    systemctl enable --now docker
    usermod -aG docker vagrant

    echo ">>> Ouverture des ports firewall"
    if systemctl is-active --quiet firewalld; then
      firewall-cmd --permanent --add-port=8080/tcp
      firewall-cmd --permanent --add-port=8081/tcp
      firewall-cmd --permanent --add-port=8082/tcp
      firewall-cmd --permanent --add-port=8083/tcp
      firewall-cmd --permanent --add-port=11112/tcp
      firewall-cmd --permanent --add-port=2762/tcp
      firewall-cmd --permanent --add-port=2575/tcp
      firewall-cmd --reload
    fi
  SHELL
  
  config.vm.provision "shell", name: "write-compose-file", inline: <<-SHELL
    set -euo pipefail
    mkdir -p /opt/pacs-yata/{ldap-config,ldap-data,db-data,storage,meddream-config}
    cd /opt/pacs-yata

    cat > docker-compose.yml <<'COMPOSE_EOF'
version: "3.8"

networks:
  pacs-net:
    driver: bridge

volumes:
  ldap-data:
  ldap-config:
  db-data:
  arc-storage:

services:

  ldap:
    image: dcm4che/slapd-dcm4chee:2.6.3-29.0
    container_name: pacs-ldap
    hostname: ldap
    networks:
      - pacs-net
    ports:
      - "389:389"
    environment:
      STORAGE_DIR: /storage/fs1
    volumes:
      - ldap-data:/var/lib/openldap/openldap-data:Z
      - ldap-config:/etc/openldap/slapd.d:Z
    restart: unless-stopped

  db:
    image: dcm4che/postgres-dcm4chee:14.5-29
    container_name: pacs-db
    hostname: db
    networks:
      - pacs-net
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: pacsdb
      POSTGRES_USER: pacs
      POSTGRES_PASSWORD: Med34305511
    volumes:
      - db-data:/var/lib/postgresql/data:Z
    restart: unless-stopped

  arc:
    image: dcm4che/dcm4chee-arc-psql:5.29.0
    container_name: pacs-arc
    hostname: arc
    depends_on:
      - ldap
      - db
    networks:
      - pacs-net
    ports:
      - "8080:8080"
      - "8443:8443"
      - "11112:11112"
      - "2762:2762"
      - "2575:2575"
    environment:
      LDAP_URL: ldap://ldap:389
      LDAP_BASE_DN: dc=dcm4che,dc=org
      LDAP_ADMIN_DN: cn=admin,dc=dcm4che,dc=org
      LDAP_ADMIN_PASSWORD: admin
      POSTGRES_HOST: db
      POSTGRES_DB: pacsdb
      POSTGRES_USER: pacs
      POSTGRES_PASSWORD: Med34305511
      WILDFLY_CHOWN: "/opt/wildfly/standalone /storage"
      WILDFLY_WAIT_FOR: "ldap:389 db:5432"
    volumes:
      - arc-storage:/storage:Z
    restart: unless-stopped

  token-service:
    image: meddream/token-service:2.1.0
    container_name: pacs-token-service
    hostname: token-service
    depends_on:
      - arc
    networks:
      - pacs-net
    ports:
      - "8081:8088"
    environment:
      TOKEN_LIFETIME: "3600"
    restart: unless-stopped

  viewer-integration:
    image: meddream/dicom-viewer-integration:0.5
    container_name: pacs-viewer-integration
    hostname: viewer-integration
    depends_on:
      - arc
      - token-service
    networks:
      - pacs-net
    ports:
      - "8082:80"
    environment:
      DCM4CHEE_ARC_URL: http://arc:8080/dcm4chee-arc
      TOKEN_SERVICE_URL: http://token-service:8081
    restart: unless-stopped

  meddream-viewer:
    image: meddream/dcm4chee-dicom-viewer:8.2.0
    container_name: pacs-meddream-viewer
    hostname: meddream-viewer
    depends_on:
      - viewer-integration
      - token-service
    networks:
      - pacs-net
    ports:
      - "8083:8080"
    environment:
      DCM4CHEE_ARC_URL: http://arc:8080/dcm4chee-arc
      VIEWER_INTEGRATION_URL: http://viewer-integration:8082
      TOKEN_SERVICE_URL: http://token-service:8081
      
    volumes:
      -  /opt/pacs-yata/meddream-config/application.properties:/opt/meddream/application.properties:Z
      - arc-storage:/storage:Z 
    restart: unless-stopped
COMPOSE_EOF

    echo ">>> docker-compose.yml écrit dans /opt/pacs-yata/"
  SHELL
  
  config.vm.provision "shell", name: "write-meddream-config", inline: <<-SHELL
    set -euo pipefail
    mkdir -p /opt/pacs-yata/meddream-config

    cat > /opt/pacs-yata/meddream-config/application.properties <<'PROPS_EOF'
server.port=8080
com.softneta.license.licenseFileLocation=./license
com.softneta.meddream.loginEnabled=true
spring.profiles.include=auth-inmemory
authentication.inmemory.users[0].userName=admin
authentication.inmemory.users[0].password=admin
authorization.users[0].userName=admin
authorization.users[0].role=ADMIN,SEARCH,PATIENT_HISTORY,UPLOAD_DICOM_LIBRARY,EXPORT_ISO,EXPORT_ARCH,FORWARD,REPORT_UPLOAD,DOCUMENT_VIEW,FREE_DRAW_EDIT,SMART_DRAW_EDIT,CLEAR_CACHE,USER_SETTINGS 

com.softneta.meddream.pacs.configurations[0].type=Dcm4chee5
com.softneta.meddream.pacs.configurations[0].id=PACS
com.softneta.meddream.pacs.configurations[0].url=jdbc:postgresql://db:5432/pacsdb
com.softneta.meddream.pacs.configurations[0].username=pacs
com.softneta.meddream.pacs.configurations[0].password=Med34305511
com.softneta.meddream.pacs.configurations[0].storage=fs1=/storage/fs1/
com.softneta.meddream.pacs.configurations[0].storeScpAet=DCM4CHEE
com.softneta.meddream.pacs.configurations[0].storeScpIp=arc
com.softneta.meddream.pacs.configurations[0].storeScpPort=11112
com.softneta.meddream.pacs.configurations[0].pacsVersion=5.29
PROPS_EOF

    echo ">>> application.properties écrit pour MedDream"
  SHELL
  
  config.vm.provision "shell", name: "start-stack", run: "always", inline: <<-SHELL
    set -euo pipefail
    cd /opt/pacs-yata
    docker compose up -d
    echo ">>> Stack PACS démarrée."
  SHELL


end
