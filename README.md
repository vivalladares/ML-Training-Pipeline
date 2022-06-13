# ML-Training-Pipeline

## Preparando ambiente Apache Airflow com docker compose

```bash
$ mkdir apache-airflow
$ cd apache-airflow
$ python3 -m venv .venv
$ source .venv/scripts/activate
$ ./codeartifact_login.sh
$ python3 -m pip install --upgrade pip
$ pip install -r requirements.txt
```
Para implantar o Airflow no Docker Compose, deve-se buscar [docker-compose.yaml](https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml)
```bash
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.3.2/docker-compose.yaml'
```
Este arquivo contém várias definições de serviço:

- Airflow-scheduler: o agendador monitora todas as tarefas e DAGs e, em seguida, aciona as instâncias de tarefas quando suas dependências são concluídas
- Airflow-webserver: o servidor web está disponível em http://localhost:8080
- Airflow-worker: o trabalhador que executa as tarefas dadas pelo escalonador
- Airflow-init: o serviço de inicialização
- Postgres: o banco de dados
- Redis: o broker que encaminha mensagens do agendador para o trabalhador

Configurando o usuário correto do Airflow

Alguns diretórios no contêiner são montados, o que significa que seu conteúdo é sincronizado entre seu computador e o contêiner.

- ./dags- você pode colocar seus arquivos DAG aqui
- ./logs- contém logs de execução de tarefas e agendador
- ./plugins- você pode colocar seus plugins personalizados aqui

```bash
mkdir -p ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

Inicialize o banco de dados
Em todos os sistemas operacionais , você precisa executar migrações de banco de dados e criar a primeira conta de usuário. Para fazer isso, corra.
```bash
docker-compose up airflow-init
```

Agora você pode iniciar todos os serviços:
```bash
docker-compose up
```

## Validação de dados

A etapa de validação de dados é necessária antes do treinamento do modelo para decidir se você pode treinar o modelo ou interromper a execução do pipeline. 

- Desvios do esquema de dados : esses desvios são considerados anomalias nos dados de entrada, o que significa que as etapas do pipeline downstream, incluindo processamento de dados e treinamento de modelo, recebem dados que não estão em conformidade com o esquema esperado. As distorções de esquema incluem receber recursos inesperados, não receber todos os recursos esperados ou receber recursos com valores inesperados.
- Desvio de valores de dados : esses desvios são alterações significativas nas propriedades estatísticas dos dados, o que significa que os padrões de dados são significativamente alterados e você precisa verificar a natureza dessas alterações.

**1. CheckOperator**

O CheckOperator espera uma consulta SQL que retornará uma única linha. Cada valor nessa primeira linha é avaliado usando python bool casting. Se algum dos valores retornar False, a verificação falhará e apresentará erros. Então, basta adicionar uma tarefa ao pipeline, podemos verificar se existem dados, por exemplo, para uma data específica.
```bash
check_interaction_data = CheckOperator(
    task_id='check_interaction_data',
    sql='SELECT COUNT(1) FROM interaction WHERE interaction_date = CURRENT_DATE',
    conn_id=CONN_ID
)
```

**2. IntervalCheckOperator**

Verifica se os valores das métricas fornecidas como expressões SQL estão dentro de uma certa tolerância daqueles de “ days_back ” anteriores, permitindo que se verifique diferentes métricas ao mesmo tempo com diferentes proporções para alguma tabela específica.
- max_over_min: max(cur, ref) / min(cur, ref)
- parent_diff : calcula abs(cur-ref)/ref

```bash
check_interaction_intervals = IntervalCheckOperator(
    task_id='check_interaction_intervals',
    table='interaction',
    metrics_thresholds={'COUNT(*)': 1.5,
                        'MAX(amount)': 1.3,
                        'MIN(amount)': 1.4,
                        'SUM(amount)': 1.3},
    date_filter_column='interaction_date',
    days_back=5,
    conn_id=CONN_ID
)
```

**3. ValueCheckOperator**

Executa uma verificação de valor puro usando código SQL. Você pode usar uma consulta SQL de qualquer complexidade para obter qualquer valor. A verificação será aprovada se o valor calculado for igual ao valor passado com alguma tolerância.
```bash
check_interaction_amount_value = ValueCheckOperator(
    task_id='check_interaction_amount_value',
    sql="SELECT COUNT(1) FROM interaction WHERE interaction_date=CURRENT_DATE - 1",
    pass_value=200,
    tolerance=0.2,
    conn_id=CONN_ID
)
```
## Validação do modelo


