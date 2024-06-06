## Openshift with WatsonX and PosgreSQL for RAG

**Introduction**

This repository demonstrates the implementation of Retrieval Augmented Generation (RAG) using LangChain, WatsonX, and PostgreSQL in Openshift. RAG is a versatile pattern that enables factual recall of information, such as querying a knowledge base in natural language.

**Components**

* Openshift: Cluster Enviroment
* LangChain: A Python library for building and integrating AI models
* WatsonX: A cloud-based AI platform for building and deploying AI models
* PostgreSQL: A relational database management system with vector database capabilities using PgVector

**Demo Overview**

This demo showcases the following components:

1. Document loading and data preparation using LangChain
2. Building a knowledge using LangChain and WatsonX
3. Creating a vector store using PostgreSQL and PgVector
4. Implementing RAG using LangChain and WatsonX
5. Querying the knowledge using natural language

**Setup**

To run this demo, you'll need to:

1. Create a Openshift instance with Elyra
2. Create a server Postgre [here](./pgvector/README.md)
3. Create a Watson Machine Learning (WML) Service instance
4. Install the required dependencies, including LangChain, WatsonX, and PgVector
5. Set up a PostgreSQL database with PgVector enabled
6. Load the demo data into the database


**Running the Demo**

To run the demo, simply execute the provided Python script. The script will guide you through the RAG process, from document loading to querying the knowledge base.

**Note**

This demo is for educational purposes only and is not intended for production use. Please ensure you have the necessary permissions and licenses to use the required components.

**License**

This repository is licensed under the Apache 2.0 License. See the LICENSE file for details.


