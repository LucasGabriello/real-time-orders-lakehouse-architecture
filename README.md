
# Real-Time Orders Data Platform

## Overview

Este projeto apresenta o desenho e a implementação de uma arquitetura moderna de dados para ingestão e processamento **near real-time** de pedidos de um e-commerce.

A solução foi construída utilizando **Databricks na Azure** e segue o padrão **Medallion Architecture (Bronze, Silver, Gold)**, garantindo escalabilidade, governança e qualidade dos dados.

O objetivo é disponibilizar dados confiáveis e atualizados para times de produto, engenharia e negócio, permitindo tomadas de decisão mais rápidas e orientadas por dados.

---

## Arquitetura da Solução

![Arquitetura](./architecture/architecture.png)

---

## Objetivos da Arquitetura

- Permitir ingestão incremental de dados  
- Garantir rastreabilidade e qualidade  
- Separar responsabilidades por camada  
- Facilitar reprocessamentos  
- Preparar dados para consumo analítico  
- Simular um pipeline real-time aderente a cenários corporativos  

---

## Stack Tecnológica

| Tecnologia | Justificativa |
|------------|----------------|
| Databricks | Plataforma unificada para engenharia e analytics |
| Delta Lake | ACID transactions, schema enforcement e alta performance |
| Unity Catalog / Volumes | Governança, segurança e controle de acesso |
| PySpark | Processamento distribuído e escalável |
| Structured Streaming | Ingestão incremental com baixa latência |
| Python | Linguagem amplamente adotada no ecossistema de dados |
| Matplotlib | Visualizações simples e eficientes |

---

## Estrutura do Repositório

```

.
├── architecture
│   └── architecture.png
│
├── dataset
│   └── orders.csv
│
├── notebooks
│   ├── create_file_system
│   ├── 01_event_producer
│   ├── 02_bronze_ingestion
│   ├── 03_silver_transformation
│   ├── 04_gold_aggregation
│   └── 05_gold_analytics
│
└── README.md

```



## Fluxo de Dados

```

Dataset → Landing → Bronze → Silver → Gold → Analytics

```

### Landing
Responsável por receber os arquivos simulando eventos em tempo real.

### Bronze
Armazena os dados crus exatamente como recebidos, garantindo fidelidade à origem e permitindo reprocessamentos.

### Silver
Camada de tratamento onde são aplicadas regras de qualidade, remoção de duplicidades e validações.

### Gold
Camada de negócio com métricas prontas para consumo analítico.

---

## Dataset

Fonte:

https://www.kaggle.com/datasets/gabrielramos87/an-online-shop-business

O dataset contém dados transacionais com informações como:

- País  
- Produto  
- Quantidade  
- Preço  
- Cliente  
- Data da transação  

---

## Setup no Databricks

### 1. Criação do Volume

A solução utiliza **Unity Catalog Volumes** para garantir governança e segurança no armazenamento.

Exemplo:

```

Catalog: workspace
Schema: default
Volume: orders_volume

```

---

### 2. Estrutura de Diretórios

```

/Volumes/workspace/default/orders_volume/
landing/
bronze/
silver/
gold/
checkpoints/
source/

```

---

### 3. Upload do Dataset

Enviar o arquivo `orders.csv` para:

```

source/

````

---

## Simulação de Ingestão Near Real-Time

Foi implementado um **event producer** responsável por simular a chegada contínua de dados.

### Estratégia adotada:

- Divisão do dataset em pequenos arquivos  
- Envio periódico para a pasta `landing`  
- Permitir ingestão incremental pelo Auto Loader  

Essa abordagem possibilita demonstrar um comportamento próximo ao de sistemas baseados em eventos sem a necessidade de infraestrutura adicional como Kafka ou Event Hub.

---

## Bronze Layer

### Responsabilidades

- Ingestão incremental  
- Preservação dos dados originais  
- Inclusão de metadados técnicos  
- Base para reprocessamentos  

### Exemplo de leitura

```python
bronze_df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("header", "true")
    .schema(schema)
    .load("/Volumes/workspace/default/orders_volume/landing")
)
````

### Escrita em Delta

```python
bronze_df.writeStream \
    .format("delta") \
    .option("checkpointLocation", "/Volumes/.../checkpoints/bronze") \
    .outputMode("append") \
    .start("/Volumes/.../bronze")
```

---

## Silver Layer

### Transformações aplicadas

* Remoção de registros com quantidade ou preço inválidos
* Deduplicação por identificador de transação
* Padronização dos dados

### Exemplo

```python
silver_df = (
    spark.readStream
    .format("delta")
    .load("/Volumes/.../bronze")
    .filter("Quantity > 0")
    .filter("Price > 0")
    .dropDuplicates(["TransactionNo"])
)
```

---

## Gold Layer

Camada responsável por disponibilizar métricas consolidadas para consumo analítico.

### Métrica implementada

**Receita total por país**

```python
gold_df = (
    silver_df
    .withWatermark("ingestion_time", "10 minutes")
    .withColumn("revenue", col("Price") * col("Quantity"))
    .groupBy("Country")
    .agg(sum("revenue").alias("total_revenue"))
)
```

O uso de watermark permite controlar o crescimento do estado em agregações streaming e melhora a eficiência do processamento.

---

## Camada de Analytics

Foi desenvolvido um notebook dedicado para exploração da camada Gold, contendo visualizações e análises de negócio.

### Exemplos de análises

* Receita por país
* Top mercados
* Receita total consolidada

Essas informações auxiliam na identificação de concentração de receita e possíveis oportunidades de expansão.

---

## Boas Práticas Aplicadas

**Medallion Architecture**
Separação clara entre ingestão, tratamento e consumo.

**Delta Lake**
Consistência transacional e suporte a reprocessamentos.

**Desacoplamento de camadas**
Cada camada lê diretamente da anterior, reduzindo dependências.

**Governança**
Uso de Unity Catalog para controle e organização dos dados.

**Escalabilidade**
Processamento distribuído com Apache Spark.

---

## Possíveis Evoluções

* Integração com Kafka ou Event Hub
* Implementação de Change Data Capture (CDC)
* Orquestração com Databricks Workflows
* Monitoramento e alertas
* Data Quality com regras automatizadas
* CI/CD para pipelines de dados
* Criação de dashboards executivos

---

## Ordem de Execução

1. Setup do ambiente
2. Execução do Event Producer
3. Ingestão Bronze
4. Transformações Silver
5. Agregações Gold
6. Análises

---

## Resultados Esperados

A arquitetura proposta permite:

* Monitoramento quase em tempo real
* Dados confiáveis para análises
* Facilidade de expansão
* Governança desde a origem
* Separação clara de responsabilidades

---

## Decisões de Arquitetura

**Por que Medallion Architecture?**
Reduz a complexidade do pipeline e melhora a confiabilidade dos dados.

**Por que Delta Lake?**
Evita problemas comuns de data lakes tradicionais, oferecendo transações e controle de schema.

**Por que ingestão incremental?**
Diminui latência e custo computacional em comparação a cargas completas.

---

## Autor

Projeto desenvolvido como case técnico para demonstrar competências em:

* Engenharia de Dados
* Arquitetura de Dados
* Processamento distribuído
* Streaming
* Modelagem analítica

