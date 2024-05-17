# Encryption-and-Authentication-in-Prometheus
First: Encryption to ensure that data transmitted over the network remains confidential
1) I’m using open SSL to generate self-signed certificates.
   On the node exporter:
   
       $ sudo openssl req   -x509   -newkey rsa:4096   -nodes   -keyout node_exporter.key   -out node_exporter.crt
2) Create a folder in node_exporter in /etc
   
       $ sudo mkdir /etc/node_exporter
3) Move the generated keys to the above folder
   
       $ sudo mv node_exporter.* /etc/node_exporter/
4) Create a config.yml file with the below content in the same directory
   
       tls_server_config:
        cert_file: node_exporter.crt
        key_file: node_exporter.key
5) Change the ownership of the dirctory and files to the node_exporter user and group (which was created during the installation)
   
       $ sudo chown -R node_exporter:node_exporter /etc/node_exporter
6) Update the systemd service of node_exporter with TLS config as below
   
       $ sudo vi /etc/systemd/system/node_exporter.service
   
       [Unit]
       Description=Node Exporter
       Wants=network-online.target
       After=network-online.target
       [Service]
       User=node_exporter
       Group=node_exporter
       Type=simple
       ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/config.yml"
       [Install]
       WantedBy=multi-user.target
8) Reload the daemon and restart the node_exporter
   
       $ sudo systemctl daemon-reload
       $ sudo systemctl restart node_exporter
9) To check use curl with -k option to allow insecure connection
   
       $ curl -k https://localhost:9100/metrics
   
11) now update the prometheus configuration on its server by copying the node_exporter.crt file from the node exporter server to the Prometheus server at /etc/prometheus and update the permission to the CRT file
    
        $ sudo chown prometheus:prometheus node_exporter.crt
12) then on /etc/prometheus/prometheus.yml file
    
        - job_name: 'Linux Server'
          scheme: https
          tls_config:
            ca_file: /etc/prometheus/node_exporter.crt
            insecure_skip_verify: true
          static_configs:
           - targets: ['192.168.44.130:9100']
          
13) then restart prometheus service
    
        $ sudo systemctl restart prometheus
The second step is the authentication between prometheus server and its target.

1) I will use apache2-utils to create hash

        $ Install apache2-utils
        $ sudo apt-get update && sudo apt install apache2-utils -y
   
2) Run the below command to create hash, which will prompt for a password and I used 'menna' as a password
   
        $ htpasswd -nBC 12 "" | tr -d ':\n'
                                                        
3) Go back to your node exporter server, and update the config.yml file at /etc/node_exporter and use the username as 'prometheus' and the hash password that you got from the previous command
   
        tls_server_config:
          cert_file: node_exporter.crt
          key_file: node_exporter.key
        basic_auth_users:
          prometheus: $2y$12$D4GGeQvFZxjSc.XUbAm9kOlVhw7mGvFK7YqAvaVSi6g1nuPxMQ2qC
3) Restart the node_exporter
   
        $ sudo systemctl restart node_exporter
4) Next, we update Prometheus configuration with username and password at prometheus.yml file
                                                            
        - job_name: 'Linux Server'
           scheme: https
           basic_auth:
             username: prometheus
             password: menna
          tls_config:
             ca_file: /etc/prometheus/node_exporter.crt
             insecure_skip_verify: true
         static_configs:
           - targets: ['192.168.44.130:9100']

5) Restart the Prometheus server

        $ sudo systemctl restart prometheus                                                     
