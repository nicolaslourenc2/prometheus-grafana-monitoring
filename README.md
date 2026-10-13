# prometheus-grafana-monitoring

# Network & Host Monitoring Stack    

Uma stack de monitoramento self-hosted baseada em Docker, desenvolvida para coletar, armazenar e visualizar métricas de rede e hardware. O projeto utiliza Prometheus para coleta de dados e Grafana para visualização, com foco na facilidade de implantação via Docker Compose.    

## Arquitetura e Tecnologias

* **Grafana:** Plataforma de visualização de dados (porta `3000`).   
* **Prometheus:** Banco de dados de séries temporais e motor de coleta de métricas (porta `9090`).  
* **Node Exporter:** Agente que exporta as métricas de hardware e do sistema operacional da máquina host.   

## Estrutura de Volumes e Persistência     

Para garantir que as configurações e os históricos de métricas não sejam perdidos quando os containers forem recriados, a persistência de dados foi configurada em diretórios locais (bind mounts) relativos ao `docker-compose.yml`:

* `./data/grafana:/var/lib/grafana` - Armazena painéis, usuários, plugins e configurações internas do Grafana.    
* `./data/prometheus_data:/prometheus` - Armazena o banco de dados de séries temporais (TSDB) do Prometheus.    
* `./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro` - Mapeia o arquivo de configuração do Prometheus em modo somente leitura (Read-Only).     

> **Nota:** A pasta `data/` e o arquivo local `mktxp/mktxp.conf` estão no `.gitignore` e não são versionada no repositório para não expor arquivos sensiveis. O Docker criará essa pasta automaticamente na primeira execução e você só precisará atualizar suas credenciais no arquivo .conf.    

## Detalhes Técnicos de Rede    

O monitoramento local exige que o Prometheus (isolado na rede virtual do Docker) consiga acessar as interfaces físicas da máquina hospedeira. Para solucionar isso:     
1. O **Node Exporter** está configurado com `network_mode: host` e `pid: host`, rodando diretamente na interface de rede da máquina e escutando na porta `9100`.    
2. O **Prometheus** utiliza a diretiva `extra_hosts: - "host.docker.internal:host-gateway"` para conseguir resolver o endereço da máquina hospedeira e coletar as métricas através do alvo `host.docker.internal:9100`.     
3. O **mktxp** roda isolado na rede virtual do Docker, realizando chamadas contínuas à API da RouterBoard na porta `8728` e expondo os resultados na porta `49090`.

## Como executar

1. Clone o repositório:
```
    git clone (https://github.com/nicolaslourenc2/prometheus-grafana-monitoring)
    cd prometheus-grafana-monitoring
```

2. Configure as credenciais do MikroTik:
```
# Copie o template de exemplo para criar o arquivo real de configuração local
cp mktxp/mktxp.example.conf mktxp/mktxp.conf
```

3. Inicie os containers em segundo plano:
```
    docker compose up -d
```
Abra o arquivo mktxp/mktxp.conf e substitua o IP e as credenciais (username e password) pelos dados reais da sua RouterBoard.   

4. Acesse os serviços nos navegadores:
```
    Grafana: http://localhost:3000 (Importe o Dashboard MikroTik ID: 13679)
    Prometheus: http://localhost:9090        
    Node Exporter (Métricas brutas): http://localhost:9100/metrics
    mktxp (Métricas brutas): http://localhost:49090/metrics
```

