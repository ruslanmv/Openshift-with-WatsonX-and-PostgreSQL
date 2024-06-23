# How to Perform RAG with WatsonX.ai, OpenShift AI using PostgreSQL pgvector

In this tutorial, we will guide you through the steps of setting up a Retrieval-Augmented Generation (RAG) system using IBM WatsonX.ai on OpenShift AI using PostgreSQL and pgvector. We will cover the following steps:

1. Creation of an OpenShift cluster with OpenShift AI (JuypterLab, Python) running on it. 
2. Creation of the PostgreSQL server with pgvector
3. Creation of the notebook for setting up the vector database
4. Creation of the notebook for implementing RAG with WatsonX and PostgreSQL
5. A simple demo of execution
6. Conclusion

## 1. Creation of the OpenShift cluster with OpenShift AI (JupyterLab and Python 3.11)

 Follow these steps:

1. **Create an OpenShift cluster**: Using https://docs.openshift.com/container-platform/4.15/installing/index.html create an OpenShift cluster (either on prem or on public cloud)

2. **Install OpenShift AI**: In the OpenShift cluster, navigate to Operator hub and install OpenShift AI operator and install it.


3. **Deploy JupyterLab**: Deploy JupyterLab with a custom Python 3.11 image.
 https://github.com/opendatahub-io-contrib/workbench-images?tab=readme-ov-file#workbench-images-1 has a list of additional workbench images that are available for OpenShift AI.
Import Jupyter Langchain (CUDA) image
```
   quay.io/opendatahub-contrib/workbench-images:cuda-jupyter-langchain-c9s-py311_2023c_latest
```

   <img width="1663" alt="image" src="https://github.com/ruslanmv/Openshift-with-WatsonX-and-PostgreSQL/assets/19476054/ef849755-5279-4766-835a-35d4279c84c1">


## 2. Creation of the PostgreSQL Server with pgvector

### About pgvector

pgvector is an open-source vector similarity search extension for PostgreSQL, allowing you to store and query vectors alongside your relational data. It supports exact and approximate nearest neighbor search using various distance metrics.

### Container Image

Use the prebuilt PostgreSQL 15 + pgvector image available at [Quay.io](https://quay.io/repository/rh-aiservices-bu/postgresql-15-pgvector-c9s).

### Deployment

To deploy a PostgreSQL + pgvector instance, follow these steps:

1. **Create a Secret**: Define the database name, username, and password in `01_db_secret.yaml`.
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: db-secret
   type: Opaque
   data:
     dbname: <base64-encoded-database-name>
     username: <base64-encoded-username>
     password: <base64-encoded-password>
   ```

   Create the secret in OpenShift:
   ```bash
   oc apply -f 01_db_secret.yaml
   ```

2. **Create a Persistent Volume Claim (PVC)**: Define the PVC in `02_pvc.yaml`.
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: postgresql-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
   ```

   Apply the PVC:
   ```bash
   oc apply -f 02_pvc.yaml
   ```

3. **Deploy the PostgreSQL Server**: Define the deployment in `03_deployment.yaml`.
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: postgresql
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: postgresql
     template:
       metadata:
         labels:
           app: postgresql
       spec:
         containers:
         - name: postgresql
           image: quay.io/rh-aiservices-bu/postgresql-15-pgvector-c9s:latest
           ports:
           - containerPort: 5432
           envFrom:
           - secretRef:
               name: db-secret
           volumeMounts:
           - mountPath: /var/lib/postgresql/data
             name: postgresql-storage
         volumes:
         - name: postgresql-storage
           persistentVolumeClaim:
             claimName: postgresql-pvc
   ```

   Deploy the server:
   ```bash
   oc apply -f 03_deployment.yaml
   ```

4. **Expose the PostgreSQL Service**: Define the service in `04_services.yaml`.
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: postgresql
   spec:
     ports:
     - port: 5432
     selector:
       app: postgresql
   ```

   Expose the service:
   ```bash
   oc apply -f 04_services.yaml
   ```

After applying these files, your PostgreSQL + pgvector server should be running and accessible.

### Enabling PgVector Extension

To enable the pgvector extension, connect to the PostgreSQL server as a superuser and execute:
```bash
psql -d vectordb -c "CREATE EXTENSION vector;"
```
Replace `vectordb` with your database name.

## 3. Creation of the Notebook for Setting Up the Vector Database

Create a new Jupyter notebook and follow these steps to set up your vector database:

### Load the Credentials

```python
from dotenv import load_dotenv
import os
import psycopg2

# Load the .env file
load_dotenv()

# Get the values from the .env file, or use default values if not set
user = os.getenv("user", "testuser")
password = os.getenv("password", "testpwd")
database = os.getenv("database", "vectordb")
server = os.getenv("server", "localhost")

print("User:", user)
print("Database:", database)

# Construct the connection string
CONNECTION_STRING = f"postgresql://{user}:{password}@{server}/{database}"
print(CONNECTION_STRING)
```

### Test the Server Connection

```python
conn = psycopg2.connect(
    host=server,
    database=database,
    user=user,
    password=password
)

cur = conn.cursor()
cur.execute("SELECT 1")
print(cur.fetchone())  # Should print (1,)
conn.close()
```

### Create the Vector Database

```python
# Create a connection to the database
conn = psycopg2.connect(CONNECTION_STRING)

# Create a cursor object to execute queries
cur = conn.cursor()

# Execute the SQL command
cur.execute("""
    CREATE EXTENSION IF NOT EXISTS vector;
    CREATE TABLE IF NOT EXISTS embeddings (
      id SERIAL PRIMARY KEY,
      embedding vector,
      text text,
      created_at timestamptz DEFAULT now()
    );
""")

# Commit the changes
conn.commit()

# Close the cursor and connection
cur.close()
conn.close()
```

### Verify the Table Creation

```python
# Create a connection to the database
conn = psycopg2.connect(CONNECTION_STRING)

# Create a cursor object to execute queries
cur = conn.cursor()

# Check if the table exists
cur.execute("SELECT EXISTS (SELECT 1 FROM information_schema.tables WHERE table_name = 'embeddings')")
table_exists = cur.fetchone()[0]

if table_exists:
    print("Table 'embeddings' exists!")
else:
    print("Table 'embeddings' does not exist.")

# Get the schema of the table
cur.execute("SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'embeddings'")
schema = cur.fetchall()

print("Schema of table 'embeddings':")
for column in schema:
    print(f"  {column[0]}: {column[1]}")

# Close the cursor and connection
cur.close()
conn.close()
```

## 4. Creation of the Notebook for Implementing RAG with WatsonX and PostgreSQL

Create a new Jupyter notebook and follow these steps to implement RAG:

### Load the Required Libraries and Set Up Environment

```python
import os
from dotenv import load_dotenv
from langchain.vectorstores.pgvector import PGVector
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.document_loaders import PyPDFDirectoryLoader
from langchain.chains import RetrievalQA
from ibm_watsonx_ai import WatsonxLLM, GenParams, DecodingMethods

# Load the .env file
load_dotenv()

# PostgreSQL connection details
CONNECTION_STRING = f"postgresql://{user}:{password}@{server}/{database}"
```

### Load and Process PDF Documents

```python
import wget

pdf_folder_path = './rhods-doc'
filename = 'Vector_database.pdf'
url = 'https://github.com/ruslanmv/WatsonX-with-Langchain-PostgreSQL-with-pgvector/raw/master/rhods-doc/Vector_database.pdf'

# Create the directory if it doesn't exist
if not os.path.exists(pdf_folder_path):
    os.makedirs(pdf_folder_path)

full_path = os.path.join(pdf_folder_path, filename)

if not os.path.isfile(full_path):
    wget.download(url, out=full_path)

loader = PyPDFDirectoryLoader(pdf_folder_path)
docs = loader.load()    
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1024, chunk_overlap=40)
all_splits_pdfs = text_splitter.split_documents(docs)

print(all_splits_pdfs[0])
for doc in all_splits_pdfs:
    doc.page_content = doc.page_content.replace('\x00', '')

emb

eddings = HuggingFaceEmbeddings()
COLLECTION_NAME = "documents_test"

db = PGVector.from_documents(
    documents=all_splits_pdfs,
    embedding=embeddings,
    collection_name=COLLECTION_NAME,
    connection_string=CONNECTION_STRING,
)    
```

### Initialize WatsonxLLM and Set Up RetrievalQA

```python
model_id = 'ibm/granite-13b-chat-v2'

# WatsonxLLM initialization
parameters = {
    GenParams.DECODING_METHOD: DecodingMethods.SAMPLE.value,
    GenParams.MAX_NEW_TOKENS: 1000,
    GenParams.MIN_NEW_TOKENS: 50,
    GenParams.TEMPERATURE: 0.7,
    GenParams.TOP_K: 50,
    GenParams.TOP_P: 1
}

credentials = {
    "url": os.getenv("WATSONX_URL"),
    "apikey": os.getenv("WATSONX_API_KEY")
}

watsonx_granite = WatsonxLLM(
    model_id='ibm/granite-13b-chat-v2',
    url=credentials.get("url"),
    apikey=credentials.get("apikey"),
)

qa = RetrievalQA.from_chain_type(llm=watsonx_granite, chain_type="stuff", retriever=db.as_retriever())

query = "What is a vector database?"
print(qa.run(query))
```

### Test RAG Queries

```python
query = "What is Retrieval-Augmented Generation?"
docs_with_score = db.similarity_search_with_score(query)

for doc, score in docs_with_score[:1]:
    print("-" * 80)
    print("Score: ", score)
    print(doc.page_content)
    print("-" * 80)

qa = RetrievalQA.from_chain_type(llm=watsonx_granite, chain_type="stuff", retriever=db.as_retriever())    

query = "What is Prompt?"
print(qa.run(query))

query = "What is Retrieval-Augmented Generation?"
print(qa.run(query))
```

## 5. Simple Demo of Execution

In your Jupyter notebook, execute the cells sequentially to set up the environment, load and process the documents, initialize the WatsonxLLM, and test RAG queries. You should see the output of the queries, demonstrating the capabilities of the RAG system.

## 6. Conclusion

In this tutorial, we demonstrated how to set up a Retrieval-Augmented Generation system using IBM WatsonX.ai on OpenShift AI with PostgreSQL and pgvector. We covered the deployment of the Jupyter Langchain (CUDA) image on OpenShift AI, deployment of PostgreSQL with pgvector, setting up the vector database, and implementing RAG with WatsonX. This setup enables efficient and effective retrieval of information, enhancing the capabilities of language models with a vector database backend.
