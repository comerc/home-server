> *заменить {IP} & {username}

```yml
# docker-compose.yml
services:
  ocserv:
    image: quay.io/aminvakil/ocserv
    container_name: ocserv
    privileged: true
    restart: always
    ports:
      - "443:443"
    environment:
      - SRV_CN={IP}
      - SRV_ORG=PersonalVPN
      - CA_CN=PersonalVPNCA
      - CA_ORG=PersonalVPN
      - CA_DAYS=3650
      - SRV_DAYS=3650
      - NO_TEST_USER=1
    command: ["ocserv", "-c", "/etc/ocserv/ocserv.conf", "-f"]
```

```bash
# docker compose down
# docker system prune -af
docker pull quay.io/aminvakil/ocserv
docker compose up -d
docker exec -ti ocserv ocpasswd -c /etc/ocserv/ocpasswd {username}
```

DNS leak: https://dnscheck.tools/