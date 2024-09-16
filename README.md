# fscrawler-local-setup
fscrawler-localsetup
Please test before moving to remote folders (object storage/Sharepoint, etc)

## elser_2
setup elser_model_2 model in the ElasticSearch;

## set up ingestion
- index file definition in the `/config`
  - create an index subfolder in the `/config` : `/marekidxtxt`
  - add the `_settings.yaml` file in the `marekidxtxt` folder

```yaml
---
name: "marekidxtxt"
fs:
  url: "/tmp/es"
  update_rate: "15m"
  indexed_chars: 100%
  excludes:
  - "*/~*"
  json_support: false
  filename_as_id: false
  add_filesize: true
  remove_deleted: true
  add_as_inner_object: false
  store_source: false
  index_content: true
  attributes_support: false
  raw_metadata: false
  xml_support: false
  index_folders: true
  lang_detect: false
  continue_on_error: true
  ocr:
    language: "eng"
    enabled: true
    pdf_strategy: "ocr_and_text"
  follow_symlinks: false
elasticsearch:
  nodes:
  - url: "https://<here goes instance>.< here goes link like: .elastic-cloud.com> :< here goes port>"
  username: "elastic user"
  password: "some password"
  ssl_verification: false
```

- in the root folder
 - add files in folders to get ingested, for example: 
   -  `/marekidxtxt-data` - as a folder with your data: pdfs, docs, scans, etc
 - create `.env` file - create the file using something similar to the following:

```sh
FSCRAWLER_VERSION=latest
FSCRAWLER_PORT=8080
COMPOSE_PROJECT_NAME=fscrawler
DOCUMENTS_DIRECTORY=D:\rag\fscrawler-docker-config\marekidxtxt-data
```



## prepare ingestion

- using `docker-compse.yml`:
```sh
---
version: "2.2"

services:
  fscrawler:
      image: dadoonet/fscrawler:${FSCRAWLER_VERSION}
      container_name: fscrawler_pid
      restart: always
      volumes:
        - ${DOCUMENTS_DIRECTORY}:/tmp/es:ro
        - D:\marek\fscrawler-docker-config\config:/root/.fscrawler
        - D:\marek\fscrawler-docker-config\logs:/usr/share/fscrawler/logs
      ports:
        - ${FSCRAWLER_PORT}:8080
      command: fscrawler marekidxtxt --restart --rest
```

### on windows machine:
the correct path to `config` and `logs` can be obtained in the following way:
```
1. A terminal window opens on the lower right of your screen.
2. You should be in the fscrawler-docker-config directory. If not, change the directory usingthe cd fscrawler-docker-config command.
3. In the terminal, type pwd.
4. Copy the Path to your clipboard.
5. Paste the path into the two specified sections of the docker-compose.yml file.
```

using the previous setup the `Document Directory` is set to: `D:\marek\fscrawler-docker-config\marekidxtxt-data`

## Run the ingestion
use `docker compose up [-d]` or leverage podman dedicated command (to be provided later)

(`[-d]` stands for deattached process otherwise it is connected to the terminal)


## verify

Navigate to the Kibana console.
Navigate to Search > Content > Indices.
Open the `pididxtxt` index.
Open the Documents tab to see the data for verification.


## Create destination Index

Create a destination index using the same schema as the source index. Add a field to store the content embeddings.

Enter the following code, then hit the run icon.
```
    PUT /pididx-dest
    {
    "mappings": {
        "properties": {
        "content_embedding": { 
            "type": "sparse_vector" 
        },
        "title": { 
            "type": "text" 
        },
        "content": { 
            "type": "text" 
        },
        "source": { 
            "type": "text" 
        },
        "url": { 
            "type": "text" 
        },
        "public_record": { 
            "type": "boolean" 
        }
        }
    }
    }
```

## Ingest the Data to Generate Text Embeddings

Create an ingest pipeline with an inference processor. Enter the following code:

```
    PUT _ingest/pipeline/pid-embedding-pipeline
    {
    "processors": [
        {
        "inference": {
            "model_id": ".elser_model_2_linux-x86_64",
            "input_output": [ 
            {
                "input_field": "content",
                "output_field": "content_embedding"
            }
            ]
        }
        }
    ]
    }

```

Click the run icon.

Ingest the data through the inference index pipeline to create the text embeddings. Enter the following code into the dev console:
```
    POST _reindex?wait_for_completion=false
    {
    "source": {
        "index": "pididxtxt",
        "size": 50 
    },
    "dest": {
        "index": "pididx-dest",
        "pipeline": "pid-embedding-pipeline"
    }
    }
```
To get the name of the pipeline with the model loaded, navigate to Kibana > Machine Learning > Trained Models.
Expand the Deployed model.
Navigate to the Pipelines tab to view the my-content-embesddings-pipeline created in the above step.
To confirm the task was run successfully, run the following command using the task ID produced in the response from the previous command. GET _tasks/<task_id>.

## verify when completed = true
Verify the content embeddings are in the new destination index.
Navigate to Kibana.
Navigate to Search > Content > Indices.
Open the pididx-dest index.
Open the Documents tab to see the data. It should show text chunks and sparce embeddings (vectors)

## deleting resources after using

as previously created - now you can delete the pipeline
`DELETE _ingest/pipeline/pid-embedding-pipeline`

And using the interface for indices you might be able to delete a folder, txt based index, and the sparce vector index.








