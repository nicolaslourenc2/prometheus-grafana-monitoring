# prometheus-grafana-monitoring

Stack de monitoramento self-hosted com Docker Compose para coletar, armazenar e visualizar métricas de **roteadores MikroTik** e do **servidor host**. Usa Prometheus para coleta, Grafana para visualização e o `mktxp` para extrair métricas do RouterOS por uma **conexão criptografada (API-SSL, porta `8729`)**, com CA própria gerada no próprio MikroTik e verificação de certificado ativada.

## Destaques

- Deploy com um único `docker compose up -d`.
- Comunicação com o RouterOS via **API-SSL (8729)**, sem tráfego em texto puro.
- **CA e certificados gerados no próprio MikroTik**, com validação de certificado e de hostname no exporter.
- Suporte a **vários roteadores** com uma configuração base compartilhada (herança do `[default]`).
- Dados persistentes (Grafana e Prometheus) e segredos fora do Git.

## Arquitetura e Tecnologias

```
MikroTik(s) ──API-SSL :8729──> mktxp ──:49090──> Prometheus ──> Grafana
                                                      ▲
Servidor host ──> Node Exporter (:9100) ──────────────┘
```

- **Grafana:** visualização de dados (porta `3000`).
- **Prometheus:** banco de séries temporais e motor de coleta (porta `9090`).
- **Node Exporter:** exporta métricas de hardware e do sistema operacional do host (porta `9100`).
- **mktxp (MikroTik Exporter):** agente em Python que consulta o RouterOS pela API-SSL (porta `8729`) e expõe as métricas para o Prometheus (porta `49090`).

## Estrutura do repositório

```
.
├── docker-compose.yml
├── LICENSE
├── README.md
├── mktxp/
│   ├── mktxp.example.conf   # template versionado
│   ├── mktxp.conf           # configuração real (local, ignorado pelo Git)
│   └── ca-bundle.crt        # CA(s) dos roteadores (local, ignorado pelo Git)
├── prometheus/
│   └── prometheus.yml
└── data/                    # dados persistentes (local, ignorado pelo Git)
    ├── grafana/
    └── prometheus_data/
```

## Estrutura de Volumes e Persistência

Para que configurações e históricos não sejam perdidos quando os containers forem recriados, os dados ficam em diretórios locais (bind mounts) relativos ao `docker-compose.yml`:

- `./data/grafana:/var/lib/grafana` - painéis, usuários, plugins e configurações internas do Grafana.
- `./data/prometheus_data:/prometheus` - banco de séries temporais (TSDB) do Prometheus.
- `./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro` - configuração do Prometheus em modo somente leitura.
- `./mktxp/mktxp.conf:/etc/mktxp/mktxp.conf` - configuração e credenciais do exporter do MikroTik.
- `./mktxp/ca-bundle.crt:/etc/mktxp/ca-bundle.crt:ro` - certificado(s) da CA usados para validar a conexão TLS com os roteadores.

> **Nota:** `data/`, `mktxp/mktxp.conf` e os certificados (`mktxp/*.crt`) estão no `.gitignore` e **não são versionados**, para não expor credenciais nem informações da infraestrutura. O Docker cria a pasta `data/` automaticamente na primeira execução.

## Detalhes Técnicos de Rede

O Prometheus roda isolado na rede virtual do Docker e precisa alcançar as métricas do host. Para isso:

1. O **Node Exporter** usa `network_mode: host` e `pid: host`, rodando direto na rede da máquina e escutando na porta `9100`.
2. O **Prometheus** usa `extra_hosts: - "host.docker.internal:host-gateway"` para resolver o endereço do host e coletar o alvo `host.docker.internal:9100`.
3. O **mktxp** roda na rede virtual do Docker, consulta continuamente a API dos roteadores na porta **`8729` (TLS)** e expõe os resultados na porta `49090`.

## Segurança da conexão com o MikroTik

A comunicação entre o exporter e o RouterOS usa **API-SSL** e valida o certificado do roteador:

- `use_ssl = True`: conexão criptografada na porta `8729`.
- `ssl_certificate_verify = True`: o certificado apresentado pelo roteador precisa ser assinado pela CA configurada.
- `ssl_check_hostname = True`: o `hostname` configurado precisa corresponder ao CN/SAN do certificado.
- `ssl_ca_file = /etc/mktxp/ca-bundle.crt`: arquivo com a(s) CA(s) confiáveis, montado como somente leitura.

Recomendações do lado do MikroTik (veja a seção seguinte): usar um usuário dedicado somente leitura, desabilitar o serviço `api` (porta `8728`, sem criptografia) e restringir o acesso ao `api-ssl` apenas ao IP do servidor de monitoramento.

## Gerando a CA e o certificado no MikroTik

Comandos de exemplo para RouterOS 7, executados no terminal do roteador. Substitua os valores entre `< >` e confira a sintaxe na versão do seu RouterOS.

```routeros
# 1. Criar e assinar a CA
/certificate add name=local-ca common-name=local-ca days-valid=3650 key-usage=key-cert-sign,crl-sign
/certificate sign local-ca

# 2. Criar o certificado do serviço e assinar com a CA
#    O common-name/SAN deve ser o mesmo valor usado em "hostname" no mktxp.conf
/certificate add name=api-cert common-name=<IP-OU-HOSTNAME> subject-alt-name=IP:<IP> days-valid=3650 key-usage=digital-signature,key-encipherment,tls-server
/certificate sign api-cert ca=local-ca

# 3. Habilitar o API-SSL com o certificado, restrito ao servidor de monitoramento
/ip service set api-ssl certificate=api-cert disabled=no address=<IP-DO-SERVIDOR>/32

# 4. Desabilitar a API sem criptografia (8728)
/ip service set api disabled=yes

# 5. (Recomendado) Grupo e usuário somente leitura para o exporter
/user group add name=monitoring policy=read,api
/user add name=<USUARIO> group=monitoring password=<SENHA-FORTE>

# 6. Exportar a CA (somente o certificado público) para copiar ao servidor
/certificate export-certificate local-ca
```

Baixe o arquivo exportado do roteador (Winbox, `scp` ou FTP) e salve-o em `mktxp/`. Se monitorar **mais de um roteador**, gere a CA em cada um e concatene os certificados em um único arquivo:

```bash
cat ca-roteador1.crt ca-roteador2.crt > mktxp/ca-bundle.crt
```

Para validar a conexão antes de subir a stack:

```bash
openssl s_client -connect <IP>:8729 -CAfile mktxp/ca-bundle.crt </dev/null | grep "Verify return code"
# Esperado: Verify return code: 0 (ok)
```

> **Importante:** a validação do certificado depende do relógio do roteador. Mantenha o NTP configurado no MikroTik.

## Configuração do mktxp (múltiplos roteadores)

O arquivo `mktxp/mktxp.example.conf` usa a **herança** do mktxp: a seção `[default]` define as opções comuns (TLS, coletores habilitados, etc.) e cada roteador (`[mikrotik-1]`, `[mikrotik-2]`, ...) herda essas opções, definindo apenas o que é próprio dele (`hostname`, `username`, `password`).

```ini
[default]
    port = 8729
    use_ssl = True
    ssl_certificate_verify = True
    ssl_check_hostname = True
    ssl_ca_file = /etc/mktxp/ca-bundle.crt
    # ... coletores habilitados/desabilitados ...

[mikrotik-1]
    hostname = <IP-OU-HOSTNAME-1>
    username = <USUARIO>
    password = <SENHA>
    enabled = True

[mikrotik-2]
    hostname = <IP-OU-HOSTNAME-2>
    username = <USUARIO>
    password = <SENHA>
    enabled = True
```

Para adicionar mais um roteador, basta criar outra seção `[mikrotik-N]`. Os coletores (`health`, `interface`, `dhcp`, `wireless`, `capsman`, `firewall`, ...) podem ser ligados e desligados no `[default]` ou sobrescritos por roteador.

## Como executar

**Pré-requisitos:** Docker e Docker Compose, e o API-SSL já configurado no(s) MikroTik(s) (seção anterior).

1. Clone o repositório:

```bash
git clone https://github.com/nicolaslourenc2/prometheus-grafana-monitoring.git
cd prometheus-grafana-monitoring
```

2. Crie a configuração local a partir do template e edite com os dados reais (IP/hostname, usuário e senha de cada roteador):

```bash
cp mktxp/mktxp.example.conf mktxp/mktxp.conf
nano mktxp/mktxp.conf
```

3. Copie a(s) CA(s) exportada(s) do MikroTik para `mktxp/ca-bundle.crt` (veja a seção de certificados).

4. Inicie os containers em segundo plano:

```bash
docker compose up -d
```

5. Verifique se o exporter está coletando:

```bash
docker compose logs -f mktxp
curl -s http://localhost:49090/metrics | head
```

6. Acesse os serviços:

| Serviço | URL |
| --- | --- |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Node Exporter (métricas brutas) | http://localhost:9100/metrics |
| mktxp (métricas brutas) | http://localhost:49090/metrics |

## Configurando o Grafana

1. No primeiro acesso, entre com o usuário padrão `admin` / `admin` e **troque a senha**.
2. Em *Connections → Data sources*, adicione o Prometheus com a URL `http://prometheus:9090`.
3. Em *Dashboards → Import*, importe o dashboard **MikroTik MKTXP Exporter** (ID `13679`) e selecione o datasource criado.

## Licença

Distribuído sob a licença MIT. Veja o arquivo [LICENSE](LICENSE).
