# ♻️ Restauração do Homelab

Este documento descreve o procedimento utilizado para reconstruir o homelab após uma instalação limpa do Ubuntu Server.

> **Importante:** este documento não contém senhas, tokens, certificados privados ou outras credenciais.

---

## 1. Visão geral

Servidor:

| Item | Configuração |
|---|---|
| Hostname | `homelab` |
| IP LAN | `192.168.1.62/24` |
| Gateway | `192.168.1.1` |
| Interface Wi-Fi | `wlp2s0b1` |
| Armazenamento externo | `/mnt/storage` |
| Filesystem | EXT4 |
| Docker network | `homelab` |

Principais componentes restaurados:

- Docker
- AdGuard Home
- Caddy
- Portainer
- Uptime Kuma
- Homepage
- Netdata
- File Browser
- Syncthing
- Watchtower
- Tailscale
- CasaOS
- monitoramento do HD externo

---

# 2. Montagem do armazenamento externo

O disco externo deve ser identificado pelo UUID e não pelo nome `/dev/sdX`.

Isso evita problemas porque um dispositivo pode aparecer como `/dev/sda` em uma inicialização e `/dev/sdb` em outra.

Identifique os discos:

```bash
lsblk -f
```

Crie o ponto de montagem, se necessário:

```bash
sudo mkdir -p /mnt/storage
```

Configure `/etc/fstab` utilizando:

```text
UUID=<UUID_DO_DISCO> /mnt/storage ext4 defaults,nofail 0 2
```

Teste:

```bash
sudo mount -a
findmnt /mnt/storage
df -h /mnt/storage
```

---

Uma versão sanitizada da configuração utilizada no servidor está disponível em:

`config-examples/netplan/00-installer-config.example.yaml`

Ela deve ser copiada e ajustada com o SSID, senha e identificadores corretos do novo ambiente antes da aplicação.


# 3. Rede

O servidor utiliza endereço IPv4 estático.

Exemplo sanitizado de configuração Netplan:

```yaml
network:
  version: 2

  wifis:
    wlp2s0b1:
      access-points:
        NOME_DA_REDE:
          password: "SENHA_DA_REDE"

      dhcp4: false
      dhcp6: false
      accept-ra: false

      addresses:
        - 192.168.1.62/24

      routes:
        - to: default
          via: 192.168.1.1

      nameservers:
        addresses:
          - 192.168.1.62
```

Validar antes de aplicar:

```bash
sudo netplan generate
```

Aplicar:

```bash
sudo netplan apply
```

Verificar:

```bash
ip -4 addr show wlp2s0b1
ip route
resolvectl dns wlp2s0b1
```

A rota padrão esperada é:

```text
default via 192.168.1.1 dev wlp2s0b1
```

### Falha encontrada durante a restauração

Uma configuração incorreta do gateway como:

```text
191.168.1.1
```

impediu a criação da rota padrão.

Consequências:

- host sem Internet;
- containers sem acesso externo;
- Google OFF no Uptime Kuma;
- monitor de Internet OFF;
- monitor Tailscale OFF.

A correção para:

```text
192.168.1.1
```

restabeleceu a conectividade.

---

# 4. Docker

Após instalar o Docker, valide:

```bash
docker version
docker ps
```

Crie a rede utilizada pelos serviços:

```bash
docker network create homelab
```

Verifique:

```bash
docker network inspect homelab
```

A rede deve permitir acesso externo:

```text
Internal=false
```

---

# 5. AdGuard Home

O AdGuard Home fornece DNS para a infraestrutura.

Os registros internos apontam para:

```text
192.168.1.62
```

Exemplos:

```text
homepage.home.arpa
portainer.home.arpa
uptime.home.arpa
netdata.home.arpa
files.home.arpa
syncthing.home.arpa
```

Teste diretamente o servidor DNS:

```bash
nslookup files.home.arpa 192.168.1.62
```

---

# 6. Caddy

O Caddy funciona como reverse proxy e utiliza certificados internos.

Estrutura persistente:

```text
/opt/caddy/
├── Caddyfile
├── data/
└── config/
```

Os diretórios que contêm certificados e chaves não devem ser publicados no GitHub.

Exemplo de proxy:

```caddy
https://portainer.home.arpa {
    tls internal
    reverse_proxy portainer:9000
}
```

---

# 7. Portainer

O Portainer utiliza armazenamento persistente.

O serviço HTTP interno está disponível na porta:

```text
9000
```

Dentro da rede Docker, o Uptime Kuma pode monitorá-lo utilizando:

```text
http://portainer:9000
```

Não utilizar HTTPS nessa porta interna.

---

# 8. Uptime Kuma

O Uptime Kuma é responsável pelo monitoramento da infraestrutura.

Serviços monitorados:

- CasaOS
- Homepage
- Google
- HD externo
- servidor
- Internet
- Netdata
- Portainer
- Syncthing
- Tailscale

Após a restauração, todos os dez monitores devem apresentar estado `UP`.

---

# 9. Netdata

O Netdata monitora recursos e desempenho do servidor.

A configuração persistente deve ser restaurada antes da recriação do container.

O serviço utiliza a porta:

```text
19999
```

---

# 10. File Browser

Dados persistentes:

```text
/mnt/storage/filebrowser/
├── config/
└── database/
```

Arquivos disponibilizados:

```text
/mnt/storage → /srv
```

Porta utilizada:

```text
8081
```

## Recuperação do usuário administrador

Caso seja necessário consultar usuários, primeiro pare o container para liberar o banco:

```bash
docker stop filebrowser
```

Depois:

```bash
docker run --rm \
  -v /mnt/storage/filebrowser/database:/database \
  filebrowser/filebrowser:latest \
  users ls \
  -d /database/filebrowser.db
```

O banco não deve ser apagado para recuperar a senha.

---

# 11. Syncthing

Principais diretórios:

```text
/mnt/storage/syncthing/documents
/mnt/storage/syncthing/photos
/mnt/storage/syncthing/videos
```

Ao restaurar a configuração e identidade existentes, o Device ID pode ser preservado.

Portas utilizadas:

```text
8384
22000/TCP
22000/UDP
21027/UDP
```

---

# 12. Watchtower

O Watchtower verifica atualizações dos containers.

A instalação atual necessita de compatibilidade com a API do Docker através da variável:

```text
DOCKER_API_VERSION=1.40
```

O agendamento utilizado é:

```text
0 0 4 * * *
```

correspondendo a 04:00 UTC.

---

# 13. Tailscale

O Tailscale fornece acesso remoto privado ao servidor.

Após reinstalação:

1. instalar o Tailscale;
2. autenticar novamente o servidor;
3. verificar o novo endereço;
4. atualizar os monitores que dependam desse endereço;
5. remover posteriormente dispositivos antigos/stale da tailnet, se necessário.

Verificação:

```bash
tailscale status
```

---

# 14. CasaOS

O CasaOS deve ser instalado antes da restauração de seus dados.

Antes de substituir a instalação limpa, é recomendável criar uma cópia de segurança do estado recém-instalado.

Diretórios relevantes:

```text
/etc/casaos
/var/lib/casaos
```

Depois da restauração, verificar:

```bash
systemctl is-active casaos
systemctl is-active casaos-gateway
```

---

# 15. Monitoramento do HD externo

O estado de `/mnt/storage` é monitorado através de:

```text
/usr/local/bin/check-hd-externo.sh
```

A execução periódica é controlada por:

```text
/etc/systemd/system/check-hd-externo.service
/etc/systemd/system/check-hd-externo.timer
```

O script compara o UUID montado com o UUID esperado e envia o resultado para um monitor Push do Uptime Kuma.

> A URL Push real contém um token e nunca deve ser versionada.

---

# 16. Verificação pós-restauração

Verificar armazenamento:

```bash
findmnt /mnt/storage
df -h /mnt/storage
```

Verificar rede:

```bash
ip -4 addr show wlp2s0b1
ip route
ping -c 3 1.1.1.1
```

Verificar DNS:

```bash
nslookup files.home.arpa 192.168.1.62
```

Verificar containers:

```bash
docker ps
```

Verificar serviços:

```bash
systemctl is-active casaos casaos-gateway tailscaled
```

Verificar conectividade de um container:

```bash
docker exec uptimekuma ping -c 3 1.1.1.1
```

---

# 17. Resultado esperado

Uma restauração bem-sucedida deve apresentar:

```text
✓ armazenamento externo montado
✓ IP estático configurado
✓ rota padrão funcionando
✓ DNS interno funcionando
✓ Docker operacional
✓ containers ativos
✓ CasaOS ativo
✓ Tailscale conectado
✓ Uptime Kuma com todos os monitores UP
```

---

## Lições aprendidas

- utilizar UUID para discos persistentes;
- validar o gateway depois de alterar o Netplan;
- testar a Internet no host antes de investigar Docker;
- testar DNS diretamente contra o AdGuard;
- separar dados persistentes dos containers;
- nunca armazenar credenciais no Git;
- manter procedimentos de recuperação documentados;
- validar o ambiente após cada reinicialização.
