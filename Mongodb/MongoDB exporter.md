###### tags: `PMM` `MongoDB` 監控`

# MongoDB exporter

```bash
#!/usr/bin/env bash

ACTION=$1
echo "${ACTION}"
# echo "$1 $2 $3"
if [[ ${ACTION} != "restart" && ${ACTION} != 'uninstall' ]]; then

mkdir -p mongodb-exporter
cd mongodb-exporter

wget https://github.com/percona/mongodb_exporter/releases/download/v0.11.1/mongodb_exporter-0.11.1.linux-amd64.tar.gz
tar xvzf mongodb_exporter-0.11.1.linux-amd64.tar.gz

# ./mongodb_exporter --mongodb.uri=mongodb_exporter:password@10.1.1.121:27017/admin
mv ./mongodb_exporter /usr/local/bin/mongodb_exporter

echo "[Unit]
Description=MongoDB Exporter
User=prometheus

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mongodb_exporter --mongodb.uri=mongodb_exporter:password@localhost:27017/admin

[Install]
WantedBy=multi-user.target" > /lib/systemd/system/mongodb_exporter.service
systemctl daemon-reload
systemctl enable mongodb_exporter.service
systemctl start mongodb_exporter.service
sleep 3
systemctl status mongodb_exporter.service
fi

if [[ ${ACTION} == "restart" ]]; then
systemctl daemon-reload
systemctl restart mongodb_exporter.service
sleep 3
systemctl status mongodb_exporter.service
fi

if [[ ${ACTION} == "uninstall" ]]; then
systemctl stop mongodb_exporter.service
sleep 3
rm /lib/systemd/system/mongodb_exporter.service
systemctl daemon-reload
rm /usr/local/bin/mongodb_exporter

fi

```