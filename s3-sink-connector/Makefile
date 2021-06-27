# shell
PIP 		= $(shell which pip)
DC 			= ${shell which docker-compose}
AWS_CLI = $(shell which aws) --endpoint-url=http://localhost:4566
CURL 		= $(shell which curl)
JQ			= $(shell which jq)

# connect
CONNECT	= localhost:8083
S3_CONNECTOR_NAME = s3-sink-local
DATAGEN_CONNECTOR_NAME = datagen-local

# path
MAKE_PATH := $(patsubst %/,%,$(dir $(abspath $(lastword $(MAKEFILE_LIST)))))
CONFIG_PATH = $(MAKE_PATH)/configs
S3_CURRENT_PATH = $(shell date +'year=%Y/month=%m/day=%d/hour=%H')

install-awscli:
	$(PIP) install awscli

configure-aws-localstack:
	$(AWS_CLI) configure --profile localstack

docker-compose-up:
	$(DC) -f $(MAKE_PATH)/docker-compose.yaml up -d

create-demo-bucket:
	$(AWS_CLI) s3 mb s3://demo-bucket --profile localstack

check-connect:
	$(CURL) $(CONNECT) | $(JQ) .

deploy-connectors:
	$(CURL) -X PUT -H "Content-Type: application/json" -d @$(CONFIG_PATH)/connector-datagen-local.json $(CONNECT)/connectors/$(DATAGEN_CONNECTOR_NAME)/config | $(JQ) .
	$(CURL) -X PUT -H "Content-Type: application/json" -d @$(CONFIG_PATH)/connector-s3-sink-local.json $(CONNECT)/connectors/$(S3_CONNECTOR_NAME)/config | $(JQ) .

check-connectors:
	$(CURL) $(CONNECT)/connectors/$(DATAGEN_CONNECTOR_NAME)/status | $(JQ) .
	$(CURL) $(CONNECT)/connectors/$(S3_CONNECTOR_NAME)/status | $(JQ) .

restart-connectors:
	$(CURL) -X POST $(CONNECT)/connectors/$(DATAGEN_CONNECTOR_NAME)/restart | $(JQ) .
	$(CURL) -X POST $(CONNECT)/connectors/$(S3_CONNECTOR_NAME)/restart | $(JQ) .

restart-connectors-tasks:
	@read -p "Task id : " taskId; \
	$(CURL) -X POST $(CONNECT)/connectors/$(DATAGEN_CONNECTOR_NAME)/tasks/$$taskId/restart | $(JQ) .; \
	$(CURL) -X POST $(CONNECT)/connectors/$(S3_CONNECTOR_NAME)/tasks/$$taskId/restart | $(JQ) .

list-bucket:
	$(AWS_CLI) s3 ls s3://demo-bucket/backup/examples-s3-sink/$(S3_CURRENT_PATH)/ --profile localstack

download-object:
	@read -p "Object name : " object; \
	$(AWS_CLI) s3 cp s3://demo-bucket/backup/examples-s3-sink/$(S3_CURRENT_PATH)/$$object $(MAKE_PATH)/data/ --profile localstack

delete-bucket:
	$(AWS_CLI) s3 rb s3://demo-bucket --force --profile localstack