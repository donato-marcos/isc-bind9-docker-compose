# 🐳 ISC BIND9 DNS Server (Primary & Secondary) via Docker Compose

Este repositório contém uma solução completa para provisionar um servidor de DNS local (Autoritativo e Cache) utilizando o **ISC BIND9** dentro de containers Docker. A arquitetura conta com um servidor **Primário** e um servidor **Secundário** com transferência de zona automática configurada através de uma rede interna isolada.

## 🏗️ Arquitetura e Fluxo de Rede

O projeto utiliza uma rede interna customizada no Docker (`dns-net`) para garantir IPs estáticos aos containers, permitindo que a replicação de zonas ocorra de forma segura e previsível.

```text
       [ Máquina Cliente / Rede Externa ]
                       │
        ┌──────────────┴──────────────┐
        ▼ (Porta 8153)                ▼ (Porta 8253)
 ┌─────────────────────────────┐┌─────────────────────────────┐
 │       Host Docker           ││       Host Docker           │
 │  (Mapeamento de Portas NAT) ││  (Mapeamento de Portas NAT) │
 └──────────────┬──────────────┘└──────────────┬──────────────┘
                │ (Rede Docker: 172.20.0.0/24) │
                ▼                              ▼
   ┌─────────────────────────┐    ┌─────────────────────────┐
   │  bind9-primary          │───►│  bind9-secondary        │
   │  IP: 172.20.0.10        │    │  IP: 172.20.0.11        │
   │  (Zona Autoritativa)    │    │  (Zone Transfer - AXFR) │
   └─────────────────────────┘    └─────────────────────────┘
```

* **Rede Interna do Docker:** `172.20.0.0/24`
* **BIND9 Primário:** IP `172.20.0.10` | Porta no Host: `8153`
* **BIND9 Secundário:** IP `172.20.0.11` | Porta no Host: `8253`

## 📁 Estrutura de Diretórios

```bash
isc-bind9/
├── docker-compose.yaml
├── .env
├── primary
│   └── config
│       └── bind
│           ├── named.conf
│           ├── named.conf.local
│           ├── named.conf.logging
│           ├── named.conf.options
│           ├── rndc.key
│           └── zones
│               ├── db.0.20.172.in-addr.arpa
│               └── db.fatec.lab
└── secondary
    └── config
        └── bind
            ├── named.conf
            ├── named.conf.local
            ├── named.conf.logging
            ├── named.conf.options
            └── rndc.key
```

## 🚀 Como Executar

### 1. Pré-requisitos
* Ter o **Docker** e o **Docker Compose** instalados na máquina host.

### 2. Configuração do Ambiente (`.env`)
Certifique-se de ajustar o arquivo `.env` na raiz do projeto com as variáveis desejadas. Exemplo padrão contido no projeto:
```env
VERSION="9.20"
TZ="America/Sao_Paulo"
DOMAIN="teste.local"
DNS_NET_SUBNET="172.20.0.0/24"
PRIMARY_IP="172.20.0.10"
SECONDARY_IP="172.20.0.11"
PRIMARY_DNS_UDP_PORT="8153"
PRIMARY_DNS_TCP_PORT="8153"
PRIMARY_RNDC_PORT="953"
SECONDARY_DNS_UDP_PORT="8253"
SECONDARY_DNS_TCP_PORT="8253"
SECONDARY_RNDC_PORT="9953"
```

### 3. Subir os Containers
Execute o comando abaixo para iniciar os servidores DNS em modo de segundo plano (*detached*):
```bash
docker compose up -d
```

### 4. Verificar o Status e Healthcheck
Os containers possuem regras de *healthcheck* integradas que validam se o serviço do BIND9 está ativo e respondendo consultas localmente. Para acompanhar o status, use:
```bash
docker compose ps
```

## 🛠️ Validação e Testes Práticos

Como os servidores estão publicados temporariamente em portas de laboratório (`8153` e `8253`) para evitar conflitos na porta padrão do Host, as consultas manuais devem especificar as respectivas portas.

Substitua `<IP_DO_SERVIDOR_DOCKER>` pelo IP real da máquina onde o Docker está rodando.

### Testar Resolução no Servidor Primário (Porta 8153)
```bash
dig @<IP_DO_SERVIDOR_DOCKER> -p 8153 zabbix.fatec.lab
```

### Testar Resolução no Servidor Secundário (Porta 8253)
```bash
dig @<IP_DO_SERVIDOR_DOCKER> -p 8253 grafana.fatec.lab
```

### Validar a Sintaxe do Arquivo de Zona internamente
Caso altere as zonas e queira garantir que não há erros de digitação antes de reiniciar o container, execute:
```bash
docker compose exec bind9-primary named-checkzone fatec.lab /etc/bind/zones/db.fatec.lab
```

## 💡 Dica para "Produção" (Porta 53)
Para utilizar este servidor DNS de forma definitiva na sua rede local/pessoal:
1. Altere as variáveis `PRIMARY_DNS_UDP_PORT` e `PRIMARY_DNS_TCP_PORT` no arquivo `.env` para a porta padrão `53`.
2. Certifique-se de desativar qualquer serviço de DNS local que possa estar ocupando a porta 53 no host (como o `systemd-resolved` em algumas distribuições Linux).
3. Aponte os IPs do **Name Server (NS)** contidos dentro do arquivo de zona para o IP real da máquina física/VM do Docker.

---
✍️ Desenvolvido para fins educacionais e laboratórios de Infraestrutura de Redes.
