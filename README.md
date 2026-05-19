# Legal Eagle RAG Review

Esse repo não tem a proposta de ser muito complexo ou explicar muito coisa,
é mais um relato do meu primeiro contato "mais intenso" com a GCP, o 'Legal Eagle'
é um RAG feito em um workshop em um GDG (GDG São Paulo) em 2026.

Tive o primeiro contato com o projeto / ideia no GDG Americana & Caraguá no workshop do
Victor Pugliese e depois o projeto foi continuado pelo Pedro Lara.

# Etapas

## [O Hello World](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#1)

![Hello World](./images/hello-world.png)

Assim que criei o projeto ele entrou na telinha `Home` e acessamos pela primeira vez o terminal.
Também nos foi apresentado o editor de código embutido e os recursos que ele nos oferece.

![IDE](./images/ide.png)

## Iniciando setup do ambiente

Testamos alguns comandos e já aplicamos outros:

1. Verificamos nosso login

```shell
gcloud auth list
```

2. Confirmamos usar o projeto certo (já que haviamos ganho créditos)

```shell
gcloud config set project <YOUR_PROJECT_ID>
```

3. E a parte mais legal, configuramos as APIs e serviços que usariamos no projeto

```shell
gcloud services enable storage.googleapis.com  \
                        run.googleapis.com  \
                        artifactregistry.googleapis.com  \
                        aiplatform.googleapis.com \
                        eventarc.googleapis.com \
                        cloudresourcemanager.googleapis.com \
                        firestore.googleapis.com \
                        cloudaicompanion.googleapis.com
```

4. Copiamos o projeto do github para nossa instância

```shell
git clone https://github.com/weimeilin79/legal-eagle.git
```

## [Manipulando o código](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#3)

Nessa etapa a gente começou a trabalhar em cima do código em Python da nossa aplicação,
inclusive foi incentivado para que usássemos a IA que o GCP nos ofereceu como assistente.

Mas também nos foi apresentado o código esperado para caso desse errado, e foi nele que
foi baseado esse .md.

Exemplo de prompt que foi oferecido:

```shell
"""
Write a Python function called `ask_llm` that takes a user `query` as input. This function should use the `langchain` library to interact with a Vertex AI Gemini Large Language Model.  Specifically, it should:
1.  Create a `HumanMessage` object from the user's `query`.
2.  Create a `ChatPromptTemplate` that includes a `SystemMessage` and the `HumanMessage`. The system message should instruct the LLM to act as a helpful assistant in a courtroom setting, aiding an attorney by providing necessary information. It should also specify that the LLM should respond in a high-energy tone, using no more than 100 words, and offer a humorous apology if it doesn't know the answer.
3.  Format the `ChatPromptTemplate` with the provided messages.
4.  Invoke the Vertex AI LLM with the formatted prompt using the `VertexAI` class (assuming it's already initialized elsewhere as `llm`).
5.  Print the LLM's `response`.
6.  Return the `response`.
7.  Include error handling that prints an error message to the console and returns a user-friendly error message if any issues occur during the process.  The Vertex AI model should be "gemini-2.0-flash".
"""
```

Resultado esperado do código em `legal.py`:

```python
import os
import signal
import sys
import vertexai
import random
from langchain_google_vertexai import VertexAI
from langchain_core.prompts import PromptTemplate, ChatPromptTemplate
from langchain_core.messages import HumanMessage, SystemMessage
# Connect to resourse needed from Google Cloud
llm = VertexAI(model_name="gemini-2.0-flash")
def ask_llm(query):
    try:
        query_message = {
            "type": "text",
            "text": query,
        }

        input_msg = HumanMessage(content=[query_message])
        prompt_template = ChatPromptTemplate.from_messages(
            [
                SystemMessage(
                    content=(
                        "You are a helpful assistant, and you are with the attorney in a courtroom, you are helping him to win the case by providing the information he needs "
                        "Don't answer if you don't know the answer, just say sorry in a funny way possible"
                        "Use high engergy tone, don't use more than 100 words to answer"
                       # f"Here is some past conversation history between you and the user {relevant_history}"
                       # f"Here is some context that is relevant to the question {relevant_resource} that you might use"
                    )
                ),
                input_msg,
            ]
        )
        prompt = prompt_template.format()
        response = llm.invoke(prompt)
        print(f"response: {response}")
        return response
    except Exception as e:
        print(f"Error sending message to chatbot: {e}") # Log this error too!
        return f"Unable to process your request at this time. Due to the following reason: {str(e)}"
```

## [Setando o .env (venv)](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#4)

Subimos uma aplicação de teste usando o venv do python para testar a aplicação com:

```shell
cd ~/legal-eagle/webapp
python -m venv env
source env/bin/activate
export PROJECT_ID=$(gcloud config get project)
pip install -r requirements.txt
python main.py
```

Não é um código tão complexo, ele basicamente entra na pasta do projeto clonado,
cria e ativa o venv do python, define o projeto do gcp, instala as dependências e por fim, executa.

Foi legal que também foi apresentando para gente o menu de "Visualizar porta" para termos a preview do
que está sendo executado.

Depois a gente saiu do venv com:

```shell
deactivate
```

## [Repositório de Vetores](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#5)

A função dele é bem clara, é basicamente para previnir que as respostas do LLM sejam de conhecimento geral. A ideia de trablhar com esse repositório é para que ele tenha uma fonte de dados pré-estabelecida
e confiável.

No caso usamos o Firestore, que pra mim foi ótimo, já mexo com o Realtime Database e o Firestore há alguns bons 3/4 anos, mas sempre pelo painel do Firebase, então pra mim só mudou de interface.
Inclusive devo estar até mais acostumado que com o MongoDB.

![Firestore](./images/firestore.png)

Nesse caso da print, o meu já até está com alguns dados que iremos adicionar mais pra frente.

Acho que um detalhe legal dessa etapa depois de criar o banco foi definir um "embedding_vetor", para permitir a pesquisa vetorial na coleção:

```shell
export PROJECT_ID=$(gcloud config get project)
gcloud firestore indexes composite create \
--collection-group=legal_documents \
--query-scope=COLLECTION \
--field-config field-path=embedding,vector-config='{"dimension":"768", "flat": "{}"}' \
--project=${PROJECT_ID}
```

## [Criando o Bucket](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#6)

Já no Bucket é onde guardamos os arquivos brutos, os PDFs, na etapa anterior nós criamos o repositório de vetores, onde definimos o formato que iramos guardar os documentos.

![Bucket](./images/bucket.png)

## [Cloud Run](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#7)

### Setup dos Arquivos

Pra mim, essa foi a etapa mais legal e a que eu mais aprendi.

O Cloud Run é basicamente um container que vai rodar nossa aplicação, e a gente vai poder acessar ele pela internet. Mas com o `Eventarc` a gente consegue fazer com que ele rode apenas quando for necessário, ou seja, quando for enviado um arquivo para o bucket, por exemplo.

A ideia é que ele fique "dormindo", economizando recursos, até que precise realmente de recursos.

A gente basicamente acessa a pasta do loader no terminar, inicia os arquivos necessários como main.py,
requirements.txt, Dockerfile...

Dessa forma:

```shell
cd ~/legal-eagle
mkdir loader
cd loader
touch main.py requirements.txt Dockerfile
```

E depois disso colamos os códigos que foram passados pra gente nessas pastas do loader.

### Subindo os Arquivos para o Container

Depois de colocar os arquivos na pasta do loader, basicamente criamos a imagem do container com:

```shell
cd ~/legal-eagle/loader
python -m venv env
source env/bin/activate
pip install -r requirements.txt
```

E então depois de setar o venv, rodamos o para testar o loader localmente:

```shell
functions-framework --target process_file --signature-type=cloudevent --source main.py
```

Ainda mantendo o último terminal aberto e abrindo um novo (já que ele precisa estar rodando), exportamos alguns arquivos para o bucket com o comando:

```shell
export DOC_BUCKET_NAME=$(gcloud storage buckets list --format="value(name)" | grep doc-bucket)
gsutil cp ~/legal-eagle/court_cases/case-01.txt gs://$DOC_BUCKET_NAME/
```

Da para testar CloudEvents com curl:

```shell
curl -X POST -H "Content-Type: application/json" \
     -d "{
       \"specversion\": \"1.0\",
       \"type\": \"google.cloud.storage.object.v1.finalized\",
       \"source\": \"//storage.googleapis.com/$DOC_BUCKET_NAME\",
       \"subject\": \"objects/case-01.txt\",
       \"id\": \"my-event-id\",
       \"time\": \"2024-01-01T12:00:00Z\",
       \"data\": {
         \"bucket\": \"$DOC_BUCKET_NAME\",
         \"name\": \"case-01.txt\"
       }
     }" http://localhost:8080/
```

Isso deve ser o suficiente para fazer o upload do arquivo no bucket e processar ele. Aparecendo por fim
no Bucket e no Firestore:

![Chunks](./images/chunks.png)
![Bucket](./images/bucket.png)

## [Criando imagem do Container](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#9)

Aqui nós definimos um Dockerfile (uma imagem) para depois subirmos ele para os `Artifact Registry` (que nada mais é que um registro de imagens de containers do GCP)

Mais uma vez nos foi oferecido até um prompt para a criação desse Dockerfile com o Gemini, mas também uma alternativa em código bruto:

```shell
In the loader folder,
Generate a Dockerfile for a Python 3.12 Cloud Run service that uses functions-framework. It needs to:
1. Use a Python 3.12 slim base image.
2. Set the working directory to /app.
3. Copy requirements.txt and install Python dependencies.
4. Copy main.py.
5. Set the command to run functions-framework, targeting the 'process_file' function on port 8080
```

Em alternativa tinhámos:

```Dockerfile
# Use a Python 3.12 slim base image
FROM python:3.12-slim
# Set the working directory to /app
WORKDIR /app
# Copy requirements.txt and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# Copy main.py
COPY main.py .
# Set the command to run functions-framework
CMD ["functions-framework", "--target", "process_file", "--port", "8080"]
```

No fim subimos nosso container para o Artifact Registry:

```shell
gcloud artifacts repositories create my-repository \
    --repository-format=docker \
    --location=us-central1 \
    --description="My repository"

# Buildando a imagem
cd ~/legal-eagle/loader
export PROJECT_ID=$(gcloud config get project)
docker build -t gcr.io/${PROJECT_ID}/legal-eagle-loader .

# Enviar o container para o Artifact Registry
export PROJECT_ID=$(gcloud config get project)
docker tag gcr.io/${PROJECT_ID}/legal-eagle-loader us-central1-docker.pkg.dev/${PROJECT_ID}/my-repository/legal-eagle-loader
docker push us-central1-docker.pkg.dev/${PROJECT_ID}/my-repository/legal-eagle-loader
```

# [Criando um Trigger (ou Eventarc)](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#10)

Aqui nós configuramos o `Trigger`, ou `Eventarc`. É basicamente um serviço que vai ficar "observando" o bucket de documentos, e toda vez que um arquivo for enviado para ele, o trigger vai disparar o `Cloud Run` para processar o arquivo e salvar os dados no banco.

![Eventarc](./images/eventarc.png)

# [Subindo arquivos manualmente (teste)](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#11)

```shell
export DOC_BUCKET_NAME=$(gcloud storage buckets list --format="value(name)" | grep doc-bucket)
gsutil cp ~/legal-eagle/court_cases/case-02.txt gs://$DOC_BUCKET_NAME/
gsutil cp ~/legal-eagle/court_cases/case-03.txt gs://$DOC_BUCKET_NAME/
gsutil cp ~/legal-eagle/court_cases/case-06.txt gs://$DOC_BUCKET_NAME/
```

Acessando as logs do Cloud Run é possível ver os registros do processo:

```shell
xxx
POST200130 B8.3 sAPIs-Google; (+https://developers.google.com/webmasters/APIs-Google.html) https://legal-eagle-loader-bmngrueyta-uc.a.run.app/?__GCP_CloudEventsMode=GCS_NOTIFICATION
xxx
POST200130 B520 msAPIs-Google; (+https://developers.google.com/webmasters/APIs-Google.html) https://legal-eagle-loader-bmngrueyta-uc.a.run.app/?__GCP_CloudEventsMode=GCS_NOTIFICATION
xxx
POST200130 B514 msAPIs-Google; (+https://developers.google.com/webmasters/APIs-Google.html) https://legal-eagle-loader-bmngrueyta-uc.a.run.app/?__GCP_CloudEventsMode=GCS_NOTIFICATION
```

Esses dados, justamento por terem sido "engatilhados" e processados lá na nossa Cloud Run, nos permitem ver os documentos devidamente processados lá no Firestore:
![Firestore](./images/firestore.png)

# [O RAG](https://codelabs.developers.google.com/legal-eagle-rag/instructions?hl=pt-br#12)

Aqui usamos o LangChain para simplificar o desenvolvimento de aplicativos com tecnologia de modelos de linguagem grandes (LLMs).

A gente adicione alguns trechos nos respectivos lugares:

```python
from langchain_google_vertexai import VertexAIEmbeddings
from langchain_google_firestore import FirestoreVectorStore
```

```python
PROJECT_ID = os.environ.get("GOOGLE_CLOUD_PROJECT")  # Get project ID from env
embedding_model = VertexAIEmbeddings(
    model_name="text-embedding-004" ,
    project=PROJECT_ID,)
```

```python
COLLECTION_NAME = "legal_documents"
# Create a vector store
vector_store = FirestoreVectorStore(
    collection="legal_documents",
    embedding_service=embedding_model,
    content_field="original_text",
    embedding_field="embedding",
)
```

```python
def search_resource(query):
    results = []
    results = vector_store.similarity_search(query, k=5)

    combined_results = "\n".join([result.page_content for result in results])
    print(f"==>{combined_results}")
    return combined_results
```

```python
def ask_llm(query):
    try:
        query_message = {
            "type": "text",
            "text": query,
        }
        relevant_resource = search_resource(query)

        input_msg = HumanMessage(content=[query_message])
        prompt_template = ChatPromptTemplate.from_messages(
            [
                SystemMessage(
                    content=(
                        "You are a helpful assistant, and you are with the attorney in a courtroom, you are helping him to win the case by providing the information he needs "
                        "Don't answer if you don't know the answer, just say sorry in a funny way possible"
                        "Use high engergy tone, don't use more than 100 words to answer"
                        f"Here is some context that is relevant to the question {relevant_resource} that you might use"
                    )
                ),
                input_msg,
            ]
        )
        prompt = prompt_template.format()

        response = llm.invoke(prompt)
        print(f"response: {response}")
        return response
    except Exception as e:
        print(f"Error sending message to chatbot: {e}") # Log this error too!
        return f"Unable to process your request at this time. Due to the following reason: {str(e)}"
```

Exportamos ele como um container para o Cloud Run com um:

```shell
export PROJECT_ID=$(gcloud config get project)
docker build -t gcr.io/${PROJECT_ID}/legal-eagle-webapp .
docker tag gcr.io/${PROJECT_ID}/legal-eagle-webapp us-central1-docker.pkg.dev/${PROJECT_ID}/my-repository/legal-eagle-webapp
docker push us-central1-docker.pkg.dev/${PROJECT_ID}/my-repository/legal-eagle-webapp
```

E subimos para web app na Google Cloud:

```shell
export PROJECT_ID=$(gcloud config get project)
gcloud run deploy legal-eagle-webapp \
  --image us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/legal-eagle-webapp \
  --region us-central1 \
  --set-env-vars=GOOGLE_CLOUD_PROJECT=${PROJECT_ID}  \
  --allow-unauthenticated
```

# Resultado Final

Por fim, é possível ver a nossa aplicação rodando:

![Cloud Run](./images/cloudrun.png)
![Web App](./images/webapp.png)

# Finalização
Foi muito legal mexer de forma mais profunda com alguma Cloud, até aqui só tinha me aventurado com a
Digital Ocean, Hostiger, KingHost e hospedagens prontas como Netlify e Vercel, mas o Google Cloud me ofereceu uma experiencia bem interessante.

A experiência foi incrível e pude aprender muito sobre:

- Webhooks
- Cloud Run
- Cloud SQL
- Cloud Storage
- Firestore
- Vertex AI
- LangChain
- Docker
- Artifact Registry
- Eventarc

Gostei muito de ver tudo isso e estou ansioso para mexer ainda mais com o GCP.