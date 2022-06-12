# ML-Training-Pipeline

## Preparando ambiente Apache Airflow com docker compose

```bash
$ mkdir apache-airflow
$ cd apache-airflow
$ python3 -m venv .venv
$ source .venv/scripts/activate
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
