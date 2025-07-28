# code-engine-server

## Client Server Communication in IBM Cloud Code Engine


## 1. Create App Server

```bash
pip install -r src/requirements.txt
uvicorn --app-dir src server:app --reload
```

## 2. Create Client App
Go to [code-engine-client](https://github.com/remkohdev/code-engine-client) repo.

```bash
pip install -r src/requirements.txt
python src/client.py
  {'Hello': 'World'}
```

## 3. Call


## Todos

* Run server and client on local
* Run server and client on Code Engine
* [Optional] Containerize


